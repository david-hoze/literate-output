# bit — Documentation (Literate Programming Document)

This document contains all documentation files for the bit project:
specifications, refactoring plans, and design documents.

---

## docs/haskell-type-safety.md

**Path:** `docs/haskell-type-safety.md`

*Source file.*

```markdown
# Haskell type safety: a comprehensive guide for AI code assistants

Haskell's type system is the most powerful mainstream tool for preventing bugs at compile time. **The central philosophy is this: encode invariants in types so the compiler rejects invalid programs before they run.** This guide covers 17 interlocking techniques—from simple newtypes to type-level programming—that form a layered defense against bugs. Each technique is illustrated with concrete code examples and grounded in community best practices from Alexis King, Matt Parsons, Sandy Maguire, and production Haskell shops. An AI code assistant that masters these patterns can generate Haskell code that is correct by construction rather than correct by testing.

---

## 1. Make illegal states unrepresentable

This phrase, coined by Yaron Minsky in 2007, is the foundational principle of Haskell type design. The idea: **structure your data types so that invalid combinations of values cannot be constructed at all**. Compile-time enforcement replaces runtime validation.

A common antipattern uses `Maybe` fields that must be synchronized:

```haskell
-- BAD: shipmentInfo can be Just when status is Outstanding, or Nothing when PaidFor
data Order = Order
  { status       :: OrderStatus
  , shipmentInfo :: Maybe ShipmentInfo
  }
data OrderStatus = Outstanding | PaidFor
```

Four combinations of `(OrderStatus, Maybe ShipmentInfo)` exist, but only two are valid. The fix uses separate constructors:

```haskell
-- GOOD: invalid combinations are impossible to construct
data Order
  = OutstandingOrder OrderData
  | PaidOrder OrderData ShipmentInfo
```

Now a paid order always has shipment info, and an outstanding order never does. **No runtime check required.** The same principle applies to correlated optionals—replace `Maybe a -> Maybe b -> ...` with `Maybe (a, b) -> ...` when both must be present or absent together.

The principle extends to state machines encoded in types using GADTs (covered in section 4):

```haskell
{-# LANGUAGE GADTs, DataKinds, KindSignatures #-}

data CheckoutState (s :: *) where
  NoItems      :: CheckoutState Empty
  HasItems     :: [Item] -> CheckoutState Filled
  CardSelected :: [Item] -> Card -> CheckoutState Ready
  OrderPlaced  :: OrderId -> CheckoutState Complete

-- Transition functions enforce valid state flow:
selectCard :: Card -> CheckoutState Filled -> CheckoutState Ready
selectCard card (HasItems items) = CardSelected items card

placeOrder :: CheckoutState Ready -> IO (CheckoutState Complete)
placeOrder (CardSelected items card) = OrderPlaced <$> processPayment items card

-- placeOrder (HasItems [...]) is a TYPE ERROR — cannot skip card selection
```

Alexis King distinguishes **intrinsic** vs. **extrinsic** safety. `data OneToFive = One | Two | Three | Four | Five` is intrinsically safe—illegal values are literally unutterable. A `newtype OneToFive = OneToFive Int` with a hidden constructor is extrinsically safe—it relies on the module boundary. Prefer intrinsic safety where practical; use extrinsic safety (smart constructors + opaque types) as the fallback.

---

## 2. Newtypes prevent argument transposition bugs

When functions accept multiple parameters of the same primitive type, arguments can be silently swapped with no compile-time error. This is one of the most common bug categories in any language:

```haskell
-- BAD: both arguments are Double — easy to swap
calculateArea :: Double -> Double -> Double
calculateArea width height = width * height

result = calculateArea myHeight myWidth  -- compiles, silently wrong
```

The fix wraps each concept in a `newtype`. Newtypes have **zero runtime overhead**—GHC erases the wrapper after type checking:

```haskell
newtype Width  = Width  Double deriving (Show)
newtype Height = Height Double deriving (Show)

calculateArea :: Width -> Height -> Double
calculateArea (Width w) (Height h) = w * h

-- calculateArea (Height 10) (Width 5)
-- ERROR: Couldn't match expected type 'Width' with actual type 'Height'
```

Real-world Haskell libraries frequently make this mistake with `type` aliases, which provide **zero type safety**:

```haskell
-- BAD: type aliases are just synonyms — all interchangeable!
type WorkerId     = UUID
type SupervisorId = UUID
type ProcessId    = UUID
-- A WorkerId can be passed where SupervisorId is expected
```

Use `newtype` instead. The `GeneralizedNewtypeDeriving` extension eliminates boilerplate by reusing the wrapped type's instances:

```haskell
{-# LANGUAGE GeneralizedNewtypeDeriving #-}

newtype Dollars = Dollars { getDollars :: Int }
  deriving (Eq, Show, Num, Ord)

-- Arithmetic works without manual instances:
total = Dollars 5 + Dollars 3  -- Dollars 8
```

`DerivingVia` goes further, letting you choose *which* type's instance to reuse:

```haskell
{-# LANGUAGE DerivingVia #-}

newtype Salary = Salary Int
  deriving (Semigroup, Monoid) via (Sum Int)
```

---

## 3. Phantom types track state at the type level

A phantom type parameter appears in the type signature but not in the data constructor. It exists purely for compile-time tracking with zero runtime overhead:

```haskell
data Sanitized
data Unsanitized

newtype FormData a = FormData String  -- 'a' is phantom

userInput :: String -> FormData Unsanitized
userInput = FormData

sanitize :: FormData Unsanitized -> FormData Sanitized
sanitize (FormData s) = FormData (filter isAlpha s)

insertIntoDb :: FormData Sanitized -> IO ()
insertIntoDb (FormData s) = putStrLn ("Storing: " ++ s)

-- insertIntoDb (userInput "<script>alert('xss')</script>")
-- ERROR: Couldn't match type 'Unsanitized' with 'Sanitized'
-- The ONLY path to Sanitized is through the sanitize function.
```

The pattern works equally well for file handle permissions:

```haskell
data ReadMode
data WriteMode

newtype Handle mode = Handle FilePath

openRead  :: FilePath -> IO (Handle ReadMode)
openWrite :: FilePath -> IO (Handle WriteMode)

readData  :: Handle ReadMode -> IO String
writeData :: Handle WriteMode -> String -> IO ()

-- writeData readHandle "hello"
-- ERROR: Couldn't match type 'ReadMode' with 'WriteMode'
```

And for preventing unit confusion (the Mars Climate Orbiter problem):

```haskell
data USD
data EUR

newtype Amount a = Amount Double deriving (Show, Num)

-- (Amount 10 :: Amount USD) + (Amount 20 :: Amount EUR)
-- ERROR: Couldn't match type 'EUR' with 'USD'
```

Functions that don't care about the phantom parameter can be polymorphic over it, while functions that enforce a specific state constrain it. This gives you fine-grained compile-time control over which operations are valid in which states.

---

## 4. GADTs encode invariants in constructors

Generalized Algebraic Data Types let each constructor specify its own return type. This enables the compiler to **refine types during pattern matching**, unlocking type-safe expression trees, well-typed ASTs, and length-indexed vectors.

### Well-typed expression AST

```haskell
{-# LANGUAGE GADTs #-}

data Expr a where
  LitInt  :: Int  -> Expr Int
  LitBool :: Bool -> Expr Bool
  Add     :: Expr Int -> Expr Int -> Expr Int
  Equal   :: Expr Int -> Expr Int -> Expr Bool
  If      :: Expr Bool -> Expr a -> Expr a -> Expr a

-- The evaluator is TOTAL — no runtime type errors possible:
eval :: Expr a -> a
eval (LitInt n)    = n           -- GHC knows a ~ Int
eval (LitBool b)   = b           -- GHC knows a ~ Bool
eval (Add e1 e2)   = eval e1 + eval e2
eval (Equal e1 e2) = eval e1 == eval e2
eval (If c t f)    = if eval c then eval t else eval f

-- Ill-typed expressions are IMPOSSIBLE to construct:
-- Add (LitBool True) (LitInt 3)
-- ERROR: Couldn't match type 'Bool' with 'Int'
```

When you pattern match on `LitInt n`, GHC learns `a ~ Int` for that branch. This **type refinement** is the key GADT power—it lets `eval` return `Int` in one branch and `Bool` in another while maintaining the unified return type `a`. A type signature is required for GADT pattern-matching functions; without it, GHC cannot perform refinement.

### Length-indexed vectors

```haskell
{-# LANGUAGE GADTs, DataKinds, KindSignatures, TypeFamilies #-}

data Nat = Zero | Succ Nat

data Vector (n :: Nat) a where
  VNil  :: Vector 'Zero a
  VCons :: a -> Vector n a -> Vector ('Succ n) a

-- Total head — cannot be called on empty vectors:
vhead :: Vector ('Succ n) a -> a
vhead (VCons x _) = x
-- vhead VNil is a TYPE ERROR, not a runtime crash

type family Plus (m :: Nat) (n :: Nat) :: Nat where
  Plus 'Zero     n = n
  Plus ('Succ m) n = 'Succ (Plus m n)

vappend :: Vector m a -> Vector n a -> Vector (Plus m n) a
vappend VNil         ys = ys
vappend (VCons x xs) ys = VCons x (vappend xs ys)
```

GADTs also power the order state machine pattern from section 1, where `Order 'PaidFor` carries different data than `Order 'Outstanding`, and functions like `refundOrder :: Order 'PaidFor -> m ()` are statically guaranteed to receive only paid orders.

---

## 5. Type-level programming with DataKinds and type families

The `DataKinds` extension promotes every data type to a kind and its value constructors to type constructors. This creates a richer kind system where type-level values carry domain meaning:

```haskell
{-# LANGUAGE DataKinds #-}

data Nat = Zero | Succ Nat
-- Without DataKinds: Zero and Succ are value constructors of kind *
-- With DataKinds: 'Zero :: Nat and 'Succ :: Nat -> Nat exist at the type level
```

The tick mark `'` disambiguates promoted constructors. Without `DataKinds`, type-level "naturals" like `data Ze; data Su n` have kind `*`, meaning nonsensical types like `Vec Int Char` are well-kinded. With `DataKinds`, `'Zero :: Nat` is a distinct kind, preventing such errors.

GHC also provides built-in type-level naturals via `GHC.TypeLits`:

```haskell
import GHC.TypeLits
import Data.Proxy

-- KnownNat bridges type-level to term-level:
showNat :: forall n. KnownNat n => Proxy n -> String
showNat p = show (natVal p)
-- showNat (Proxy :: Proxy 42) ==> "42"
```

This enables **sized vectors** backed by efficient runtime representations:

```haskell
newtype SizedVector (n :: Nat) a = SizedVector (V.Vector a)

append :: SizedVector n a -> SizedVector m a -> SizedVector (n + m) a
append (SizedVector x) (SizedVector y) = SizedVector (x V.++ y)

-- Safe construction requires runtime length to match type-level size:
tryMakeSized :: forall n a. KnownNat n => V.Vector a -> Maybe (SizedVector n a)
tryMakeSized v
  | V.length v == fromIntegral (natVal (Proxy :: Proxy n)) = Just (SizedVector v)
  | otherwise = Nothing
```

**Type-safe matrix multiplication** demonstrates the power: a `Matrix m n a` multiplied by `Matrix n p a` produces `Matrix m p a`. The shared dimension `n` must match—dimension mismatches are compile-time errors, not runtime crashes.

**Closed type families** define type-level functions with ordered equations, evaluated top-to-bottom:

```haskell
type family If (c :: Bool) (t :: k) (e :: k) :: k where
  If 'True  t _ = t
  If 'False _ e = e
```

The **singleton pattern** bridges type and term levels, letting runtime values carry type-level evidence:

```haskell
data SNat :: Nat -> * where
  SZ :: SNat 'Z
  SS :: SNat n -> SNat ('S n)

replicate' :: SNat n -> a -> Vector n a
replicate' SZ     _ = VNil
replicate' (SS n) x = VCons x (replicate' n x)
```

| Extension | Purpose |
|-----------|---------|
| `DataKinds` | Promote data types to kinds; constructors become type constructors |
| `KindSignatures` | Annotate type parameters with their kind: `(n :: Nat)` |
| `TypeFamilies` | Define functions at the type level |
| `TypeOperators` | Use operators in types: `n + m` |
| `ScopedTypeVariables` | Let `forall`-bound variables scope into `where` clauses |

---

## 6. Exhaustive pattern matching catches missing cases

GHC's `-Wincomplete-patterns` (included in `-Wall`) warns when a function doesn't handle all constructors of a sum type. This is one of the most powerful features for maintaining large codebases:

```haskell
data Shape = Circle Double | Rectangle Double Double

area :: Shape -> Double
area (Circle r)      = pi * r * r
area (Rectangle w h) = w * h

-- Later, add a constructor:
data Shape = Circle Double | Rectangle Double Double | Triangle Double Double

-- GHC immediately warns:
-- Pattern match(es) are non-exhaustive
-- In an equation for 'area': Patterns not matched: Triangle _ _
```

**Every call site that handles `Shape` gets a compiler warning.** This is qualitatively superior to `if-else` chains, which give the compiler no structural knowledge about completeness.

Wildcard patterns defeat this protection and should be avoided:

```haskell
-- BAD: wildcard silently swallows new constructors
tasty :: Sweet -> Bool
tasty Cupcake = True
tasty Cookie  = True
tasty _       = False  -- Adding Cheesecake? No warning — silently "not tasty"

-- GOOD: explicit patterns for every constructor
tasty Cupcake   = True
tasty Cookie    = True
tasty Liquorice = False
tasty Raisins   = False
-- Adding Cheesecake now triggers a non-exhaustive warning
```

Use `-Werror=incomplete-patterns` in CI to turn these warnings into hard compile errors:

```cabal
ghc-options: -Wall -Werror=incomplete-patterns
```

---

## 7. Total functions eliminate runtime crashes

Partial functions—those that crash on certain inputs—are one of Haskell's worst legacy design decisions. **Every use of `head`, `tail`, `fromJust`, `read`, or `!!` is a potential runtime crash:**

| Function | Crash trigger | Safer alternative |
|----------|--------------|-------------------|
| `head` | Empty list | Pattern match or `Data.List.NonEmpty.head` |
| `tail` | Empty list | Pattern match or `Data.List.NonEmpty.tail` |
| `fromJust` | `Nothing` | `maybe`, `fromMaybe`, or pattern match |
| `read` | Parse failure | `readMaybe` from `Text.Read` |
| `!!` | Out of bounds | `Data.Vector.!?` or bounds-checked indexing |
| `maximum` | Empty list | `maximumMay` from `safe` or use `NonEmpty` |
| `Map.!` | Missing key | `Map.lookup` + pattern match or pattern guard |
| `last` | Empty list | Pattern match on reversed list, or `NonEmpty` |
| `init` | Empty list | Pattern match, or use length check first |

Replace partial patterns with total ones:

```haskell
-- BAD:
processFirst xs = doSomething (head xs)

-- GOOD:
processFirst (x:_) = doSomething x
processFirst []    = handleEmpty

-- BETTER: require non-emptiness in the type
processFirst :: NonEmpty a -> Result
processFirst (x :| _) = doSomething x  -- total, no Maybe needed
```

The **`safe` package** by Neil Mitchell provides safe variants: `headMay`, `headDef`, `headNote`, `readMay`, `atMay`, `tailSafe`, and more. Each partial function gets up to five variants covering different failure-handling strategies.

```haskell
import Safe

headMay [1,2,3]  -- Just 1
headMay []       -- Nothing
headDef 0 []     -- 0
readMay "42" :: Maybe Int  -- Just 42
readMay "abc" :: Maybe Int -- Nothing
```

**`Map.!` is a common source of runtime crashes** in code that uses `Data.Map`. Replace with `Map.lookup` and handle the `Maybe`:

```haskell
-- BAD: crashes if key is missing (even if you "know" it's there)
let lHash = lFiles Map.! p

-- GOOD: pattern guard handles both cases explicitly
[ Modified (LightFileEntry p lHash)
| p <- Set.toList intersection
, Just lHash <- [Map.lookup p lFiles]    -- fails gracefully, skips this entry
, Just rHash <- [Map.lookup p rFiles]
, lHash /= rHash
]
```

**Watch for `head` used in comparisons** — it crashes on empty input even though it "looks" safe:

```haskell
-- BAD: crashes if name is ""
| head name == '.'  = pure []

-- GOOD: total — isPrefixOf handles empty strings correctly
| isPrefixOf "." name  = pure []
```

Similarly, **`map head (group (sort xs))`** extracts unique elements from a sorted/grouped list. This uses partial `head` on each group. Replace with `nub xs` (or `Set.toList . Set.fromList` for O(n log n)):

```haskell
-- BAD: partial head on each group
uniqueTargets = map head (group (sort targets))

-- GOOD: total, clearer intent
length targets == length (nub targets)
```

A subtler danger: **`fromIntegral` silently truncates or overflows** across some type pairs. `fromIntegral (256 :: Int) :: Word8` produces `0` with no error. Write explicit, named conversion functions for numeric types.

---

## 8. Smart constructors enforce invariants at construction

The smart constructor pattern hides raw data constructors and exposes only validated construction functions. The module system is the enforcement mechanism:

```haskell
module Email (Email, mkEmail, unEmail) where  -- Email type, NOT constructor

newtype Email = Email Text

mkEmail :: Text -> Either EmailError Email
mkEmail t
  | "@" `T.isInfixOf` t && "." `T.isInfixOf` t = Right (Email t)
  | otherwise = Left InvalidEmailFormat

unEmail :: Email -> Text
unEmail (Email t) = t
```

External code cannot write `Email "not-an-email"`. The **only** way to obtain an `Email` value is through `mkEmail`, which validates the input. This moves all validation to construction time; all downstream code can trust the invariant.

Key implementation details from the Kowainik patterns handbook:

- Don't define a record field on the newtype (`newtype Email = Email { unEmail :: Text }`) because record update syntax could bypass validation
- Use a separate accessor function instead
- Hiding the constructor also prevents `Data.Coerce.coerce` from bypassing validation
- Provide an `unsafeEmail` escape hatch if needed for testing, with the `unsafe` prefix making it visible in code review

For richer error reporting, return `Either ValidationError a`:

```haskell
data ValidationError
  = WrongNumberOfGroups Int
  | InvalidGroupLength Int Text
  | InvalidCharacters (HashSet Char)

makeSerialNumber :: Text -> Either ValidationError SerialNumber
```

**PatternSynonyms** can restore pattern matching without exposing the constructor:

```haskell
module NonZero (NonZero(), pattern NonZero, nonZero) where

newtype NonZero a = UnsafeNonZero a
pattern NonZero a <- UnsafeNonZero a  -- read-only pattern

nonZero :: (Num a, Eq a) => a -> Maybe (NonZero a)
nonZero 0 = Nothing
nonZero i = Just (UnsafeNonZero i)
```

---

## 9. Refinement types and Liquid Haskell

Liquid Haskell adds **refinement types**—types decorated with logical predicates checked at compile time via an SMT solver (Z3). Specifications are written in special comments `{-@ ... @-}` so they don't affect normal compilation:

```haskell
{-@ type Pos = {v:Int | v > 0} @-}

{-@ safeDiv :: Int -> {v:Int | v /= 0} -> Int @-}
safeDiv :: Int -> Int -> Int
safeDiv x y = x `div` y

-- safeDiv 10 0  →  LIQUID TYPE ERROR at compile time
```

Liquid Haskell can prove array indices are in bounds:

```haskell
{-@ (!!) :: xs:[a] -> {n:Int | 0 <= n && n < len xs} -> a @-}
```

And verify that functions preserve invariants:

```haskell
{-@ avg :: {xs:[Int] | len xs > 0} -> Int @-}
avg :: [Int] -> Int
avg xs = safeDiv (sum xs) (length xs)
-- LiquidHaskell verifies: len xs > 0 implies length xs /= 0
```

Liquid Haskell is now available as a **GHC plugin** rather than a standalone tool. It has verified over 10,000 lines of real-world Haskell code including `containers`, `bytestring`, and `text`. The limitation is tooling maturity—it requires Z3 and can need manual annotations for complex cases. For most production Haskell, the techniques in this guide's other sections provide sufficient safety without the Liquid Haskell overhead.

---

## 10. Opaque types and module boundaries

Haskell's module system is the **only** mechanism for building abstract data types. Export the type name but not its constructors:

```haskell
module Stack (Stack, empty, push, pop, top) where  -- Stack, not Stack(..)

newtype Stack a = Stk [a]

empty :: Stack a
empty = Stk []

push :: a -> Stack a -> Stack a
push x (Stk xs) = Stk (x:xs)

pop :: Stack a -> (a, Stack a)
pop (Stk (x:xs)) = (x, Stk xs)
pop (Stk [])     = error "empty stack"
```

Since `Stk` is not exported, external code can only use the public API. The internal representation could change from `[a]` to `Vector` without affecting any consumer. This pattern is used throughout the Haskell ecosystem: `Data.Map` and `Data.Set` maintain balanced-tree invariants behind opaque constructors; `Data.Text` hides its internal `Array` representation.

The guarantee of an opaque type requires three pillars working together: **strong static types** (the compiler enforces type distinctions), **purity** (no side-channels to mutate internals), and the **module system** (hiding constructors prevents direct construction). If any pillar is missing, the guarantee breaks.

---

## 11. Purity eliminates entire bug categories

Pure functions always produce the same output for the same input and have no side effects. This makes them **trivially testable**—no mock objects, no setup, no dependency injection:

```haskell
testSort :: Bool
testSort = sort [3,1,2] == [1,2,3]  -- That's the whole test
```

**Referential transparency** means any expression can be replaced by its value. This eliminates aliasing bugs entirely—sharing references is always safe because values are immutable. It also enables **equational reasoning**: you can substitute equals for equals just like in mathematics, and prove properties of compositions like `map f . map g == map (f . g)`.

The **IO monad** enforces a strict boundary. A pure function with type `a -> b` literally cannot perform I/O—the type system prevents it:

```haskell
-- This WON'T COMPILE:
badPure :: Int -> Int
badPure x = do
    putStrLn "side effect!"  -- TYPE ERROR: IO () is not Int
    x + 1
```

**No race conditions in pure code.** Since pure functions don't access shared mutable state, they are inherently thread-safe. `parMap f xs` parallelizes a pure computation across threads with no locks, no mutexes, and no possibility of data races.

The correct architecture pushes IO to the edges: parse input in IO, process it with pure functions, then output results in IO. The pure core is where business logic lives, and it's the easiest code to test and reason about.

---

## 12. Strict data prevents space leaks

Haskell's laziness can cause **space leaks** when unevaluated thunks accumulate. The classic example:

```haskell
-- SPACE LEAK: accumulator builds up chain of unevaluated additions
badSum = go 0 [1..1000000]
  where
    go acc []     = acc
    go acc (x:xs) = go (acc + x) xs
    -- go (((0+1)+2)+3)... — millions of thunks in memory!
```

**BangPatterns** force evaluation at pattern-match time:

```haskell
{-# LANGUAGE BangPatterns #-}

goodSum = go 0 [1..1000000]
  where
    go !acc []     = acc       -- !acc forces evaluation immediately
    go !acc (x:xs) = go (acc + x) xs
    -- Constant memory: each addition evaluated before recursion
```

Measurements show this reduces memory from **~93 MB to ~5 MB** for summing one million integers.

**StrictData** makes all fields in data types strict by default within a module:

```haskell
{-# LANGUAGE StrictData #-}

-- Equivalent to: data Config = Config !Text !Int !Bool
data Config = Config
    { configName  :: Text
    , configPort  :: Int
    , configDebug :: Bool
    }
```

A subtler trap: `foldl'` is strict in the accumulator, but if the accumulator is a tuple, it only evaluates to WHNF (the tuple constructor), leaving the tuple's fields as thunks:

```haskell
-- STILL LEAKS: foldl' forces the pair, but not its contents
pairFold = foldl' (\(count, total) x -> (count+1, total+x)) (0,0) [1..1000000]

-- FIX: bang the fields
pairFold = foldl' (\(!count, !total) x -> (count+1, total+x)) (0,0) [1..1000000]
```

**Best practice**: enable `StrictData` as a default extension in your `.cabal` file. Use `~` (tilde) to opt individual fields back into laziness for infinite structures, streams, or "tying the knot" patterns.

---

## 13. Type-driven development treats the compiler as a partner

Typed holes (`_`) let you write incomplete programs and ask GHC what type goes in each gap:

```haskell
filterMap :: (a -> Maybe b) -> [a] -> [b]
filterMap f (x:xs) = case f x of
    Nothing -> _           -- GHC reports: Found hole '_' with type: [b]
    Just y  -> _           -- GHC reports: Found hole '_' with type: [b]
                           -- Relevant bindings: y :: b, f :: a -> Maybe b, xs :: [a]
```

Named holes (`_result`, `_combine`) let you track multiple unknowns. GHC reports the type and all in-scope bindings for each hole, effectively telling you what you can use to fill it.

The **type-driven development workflow**:

1. Write the type signature first
2. Stub the body with holes
3. Read GHC's reports about each hole's expected type and available bindings
4. Progressively refine, replacing holes with expressions guided by the types
5. When all holes are filled, the function type-checks

**Deferred type errors** (`-fdefer-typed-holes`) convert holes to runtime warnings, letting you compile and test other parts while leaving some functions incomplete. This is invaluable for incremental development.

GHCi provides interactive type exploration:

```
ghci> :type foldr
foldr :: Foldable t => (a -> b -> b) -> b -> t a -> b

ghci> :kind Maybe
Maybe :: * -> *

ghci> :type (>>=) @Maybe
(>>=) @Maybe :: Maybe a -> (a -> Maybe b) -> Maybe b
```

---

## 14. GHC warnings that catch real bugs

**`-Wall`** enables the most important warnings. These are the individual flags that matter most for correctness:

- **`-Wincomplete-patterns`**: Non-exhaustive pattern matches (included in `-Wall`)
- **`-Wincomplete-uni-patterns`**: Same for lambdas and pattern bindings (NOT in `-Wall`)
- **`-Wmissing-fields`**: Record construction with missing fields
- **`-Wunused-binds`**: Dead code detection
- **`-Wredundant-constraints`**: Unnecessary typeclass constraints (NOT in `-Wall`)
- **`-Wpartial-fields`**: Record fields that create partial accessor functions (NOT in `-Wall`)

Recommended production configuration in your `.cabal` file:

```cabal
ghc-options:
  -Wall
  -Wincomplete-uni-patterns
  -Wincomplete-record-updates
  -Wpartial-fields
  -Widentities
  -Wredundant-constraints
```

For CI, add `-Werror` to turn all warnings into hard compile errors. Some teams use `-Weverything` with explicit `-Wno-*` flags for the few warnings they disagree with—this ensures new GHC versions automatically enable new warnings.

---

## 15. Property-based testing leverages the type system

QuickCheck and Hedgehog generate hundreds of random test cases and automatically **shrink** failing inputs to minimal counterexamples. Haskell's type system powers this through the `Arbitrary` typeclass:

```haskell
import Test.QuickCheck

prop_reverseReverse :: [Int] -> Bool
prop_reverseReverse xs = reverse (reverse xs) == xs

-- quickCheck prop_reverseReverse
-- +++ OK, passed 100 tests.
```

**Roundtrip properties** are the most natural pattern for typed code:

```haskell
prop_jsonRoundtrip :: Person -> Bool
prop_jsonRoundtrip p = decode (encode p) == Just p

prop_showRead :: Int -> Bool
prop_showRead x = read (show x) == x
```

Custom generators give fine-grained control:

```haskell
genAge :: Gen Int
genAge = choose (0, 120)

genPerson :: Gen Person
genPerson = Person <$> genName <*> genAge
```

**Shrinking** is the killer feature: when a property fails on a large input like `[45, -12, 99, 3, -7]`, QuickCheck automatically tries smaller inputs until it finds the minimal counterexample, often something like `[0, 1]`.

**Hedgehog** improves on QuickCheck with **integrated shrinking**—generators carry shrink trees, so shrinking always respects generator invariants by construction. Hedgehog uses explicit generator values instead of a typeclass, avoiding orphan instance problems:

```haskell
import Hedgehog
import qualified Hedgehog.Gen as Gen
import qualified Hedgehog.Range as Range

prop_reverse :: Property
prop_reverse = property $ do
    xs <- forAll $ Gen.list (Range.linear 0 100) Gen.alpha
    reverse (reverse xs) === xs
```

Algebraic properties make excellent properties: commutativity, associativity, identity, idempotence, and distributivity can all be tested generically over any type with the right structure.

---

## 16. Pattern match on sum types — don't compare with `==`

Standard library types like `ExitCode`, `Ordering`, and your own sum types should be **pattern matched**, not compared with `==`. Pattern matching gives the compiler structural knowledge to check exhaustiveness and guide type refinement:

```haskell
-- BAD: boolean blindness — compiler doesn't know what True/False mean here
if code == ExitSuccess
    then doSuccess
    else doFailure

-- BAD: negated comparison is even harder to read
if code /= ExitSuccess then handleError else proceed

-- GOOD: pattern match makes the sum type explicit
case code of
    ExitSuccess -> doSuccess
    _           -> doFailure
```

This is especially important with `ExitCode` because `ExitFailure` carries an `Int` payload. Pattern matching lets you extract it; equality comparison discards it:

```haskell
-- GOOD: can inspect the failure code
case code of
    ExitSuccess      -> pure result
    ExitFailure 127  -> throwIO $ ProgramNotFound cmd
    ExitFailure n    -> throwIO $ ProcessFailed cmd n
```

The same principle applies to your own sum types. After replacing a `Bool` parameter with a sum type like `ForceMode`, use `case` rather than `==`:

```haskell
-- BAD: treats the sum type as if it were still a Bool
if fMode == Force then forcePush else normalPush

-- GOOD: pattern match — compiler checks exhaustiveness
case fMode of
    Force   -> forcePush
    NoForce -> normalPush
```

When multiple constructors share the same handler, use a **predicate + pattern guard** to collapse duplicate branches rather than duplicating the handler body:

```haskell
-- BAD: duplicated handler for two constructors
case mTarget of
    Just (TargetDevice _ _) -> filesystemAction
    Just (TargetLocalPath _) -> filesystemAction   -- exact same code
    _ -> cloudAction

-- GOOD: predicate collapses the duplicate branches
case mTarget of
    Just t | isFilesystemTarget t -> filesystemAction
    _ -> cloudAction

-- The predicate is defined once, alongside the data type:
isFilesystemTarget :: RemoteTarget -> Bool
isFilesystemTarget (TargetDevice _ _) = True
isFilesystemTarget (TargetLocalPath _) = True
isFilesystemTarget (TargetCloud _) = False
```

---

## 17. Custom sum types cure boolean blindness

The term "boolean blindness," popularized by Robert Harper, describes the problem: **a `Bool` carries no information about its provenance.** `True` from `isAdmin user` looks identical to `True` from `isRed car`—the compiler cannot distinguish them.

```haskell
-- BAD: What does True mean?
handleRequest :: Bool -> Request -> IO Response
handleRequest True  req = processRequest req
handleRequest False req = denyRequest req
-- Nothing prevents passing (isRed car) where (isAdmin user) was intended
```

The fix uses a custom sum type:

```haskell
data Access = Granted | Denied

checkAccess :: User -> Access
checkAccess user
  | isAdmin user = Granted
  | otherwise    = Denied

handleRequest :: Access -> Request -> IO Response
handleRequest Granted req = processRequest req
handleRequest Denied  req = denyRequest req
```

Sum types also scale naturally. When you add a third access level, the compiler finds every site that needs updating—something impossible with `Bool`:

```haskell
data Access = Granted | Denied | RequiresMFA
-- GHC warns everywhere Access is pattern-matched without handling RequiresMFA
```

Other examples: `data Parity = Even | Odd` instead of `isEven :: Int -> Bool`; `data IOMode = ReadMode | WriteMode | AppendMode` instead of multiple booleans; `data Keep = Keep | Drop` instead of ambiguous filter predicates.

---

## 18. Parse, don't validate

Alexis King's 2019 blog post articulated the most influential Haskell design principle of the past decade. **A parser consumes less-structured input and produces more-structured output. A validator checks a property and throws away the result.**

```haskell
-- VALIDATION: checks but discards information
validateNonEmpty :: [a] -> IO ()
validateNonEmpty (_:_) = pure ()
validateNonEmpty []    = throwIO $ userError "list cannot be empty"

-- PARSING: checks AND preserves information in the type
parseNonEmpty :: [a] -> IO (NonEmpty a)
parseNonEmpty (x:xs) = pure (x :| xs)
parseNonEmpty []     = throwIO $ userError "list cannot be empty"
```

The practical difference is dramatic. With validation, downstream code must redundantly check or use partial functions:

```haskell
-- After validation, head might still fail — the type doesn't know the list is non-empty
main = do
  dirs <- getConfigDirs          -- returns [FilePath]
  validateNonEmpty dirs
  initCache (head dirs)          -- partial! could crash if validation is removed
```

With parsing, downstream code uses total functions on refined types:

```haskell
-- After parsing, NonEmpty guarantees non-emptiness
main = do
  dirs <- parseConfigDirs        -- returns NonEmpty FilePath
  initCache (NE.head dirs)       -- total! NE.head :: NonEmpty a -> a always succeeds
```

Matt Parsons extends this in "Type Safety Back and Forth": there are two ways to make a partial function total. **Weakening the return type** (`head :: [a] -> Maybe a`) pushes burden onto callers. **Strengthening the argument type** (`head :: NonEmpty a -> a`) pushes burden onto producers. Strengthening arguments is almost always superior because it eliminates redundant checks throughout the codebase.

Parsons' "Keep Your Types Small" post makes this precise using **cardinality**: `Maybe a` has `1 + |a|` inhabitants, expanding the output space. `NonEmpty a` is a strict subset of `[a]`, restricting the input space. **Smaller types = fewer invalid states = fewer bugs.** Use `Natural` instead of `Int` when values can't be negative. Use `NonEmpty` instead of `[]` when emptiness is invalid. Use `NonZero Int` instead of `Int` for divisors.

King identifies the antipattern of **shotgun parsing**: scattering validation throughout processing code, hoping one check or another catches all bad cases. The fix is to stratify your program: **parse at the boundary** (where invalid input is rejected with structured errors), then **execute internally** (where only well-typed data flows and failure modes are minimal).

King's practical guidelines:

- Let your datatypes inform your code, not the reverse
- Don't be afraid to parse data in multiple passes
- Avoid denormalized representations (duplicated data invites inconsistency)
- Use abstract datatypes to make validators "look like" parsers when full constructive modeling isn't practical

---

## The progression of type safety techniques

These 18 techniques form a **layered defense** from simplest to most advanced. Each layer catches bugs the previous layers miss:

1. **Newtypes** — prevent argument confusion at zero runtime cost
2. **Custom sum types** — cure boolean blindness, enable exhaustive matching
3. **Pattern match on sum types** — use `case` not `==`, enabling exhaustiveness checks and payload extraction
4. **Smart constructors + opaque modules** — enforce invariants at construction time
5. **Phantom types** — track state, permissions, and units at the type level
6. **GADTs** — encode state machines and complex invariants in constructors
7. **DataKinds + type families** — type-level computation for dimensions, sizes, and indices
8. **Liquid Haskell** — SMT-verified refinement types for mathematical properties
9. **Purity + strictness** — eliminate mutation bugs and space leaks
10. **`-Wall` + totality** — catch missing cases and partial functions (including `Map.!`, `head` in guards)
11. **Property-based testing** — validate algebraic properties with automatic shrinking

An AI code assistant generating Haskell should default to these patterns: use newtypes for all domain concepts, prefer `NonEmpty` over `[]` and `Natural` over `Int`, hide constructors behind smart constructors, enable `-Wall -Wincomplete-uni-patterns -Wredundant-constraints`, avoid all partial Prelude functions (including `Map.!` and `head` in guard positions), pattern match on sum types instead of comparing with `==`, and use custom sum types instead of booleans. The result is code where the compiler catches the bugs before the tests even run.
```

---

## docs/idiomatic-haskell.md

**Path:** `docs/idiomatic-haskell.md`

*Source file.*

```markdown
# Writing idiomatic, terse Haskell: a comprehensive reference

An AI code assistant producing Haskell should default to concise, idiomatic patterns rather than verbose, beginner-style code. This means leveraging composition over explicit recursion, choosing the weakest sufficient abstraction (Functor before Applicative before Monad), using standard combinators and eliminators, and enabling the right GHC extensions. The patterns below represent community consensus drawn from the Kowainik style guide, Tibell's style guide, Tweag's guidelines, HLint rules, the STAN static analyzer, and common practice on Hackage. **Every section includes concrete before/after code examples** that demonstrate the transformation from verbose to idiomatic.

---

## Point-free style and the composition toolbox

Point-free (tacit) style defines functions without naming their arguments, building them instead by composing other functions. The canonical good case is a **simple composition chain of named functions**:

```haskell
-- Pointful
totalWordLengths s = sum (map length (words s))

-- Point-free (reads as a pipeline: words → map length → sum)
totalWordLengths = sum . map length . words
```

Point-free shines when every function in the chain is a well-known named function. It hurts when multi-argument functions force obscure combinator gymnastics — the `pointfree` tool (or Lambdabot's `@pl`) often produces unreadable output like `uncurry ((. return) . (:))`. **Rule of thumb**: if point-free requires `flip`, nested sections of `(.)`, or anonymous combinators, write it pointful.

The core operators for chaining, each with a distinct role:

| Operator | Direction | Typical use |
|---|---|---|
| `(.)` | Right-to-left | Composing pure functions: `f . g . h` |
| `($)` | Right-to-left | Final application, eliminating parens: `putStrLn $ show x` |
| `(&)` | Left-to-right | Pipeline style (from `Data.Function`): `xs & filter even & map (*2) & sum` |
| `(>>>)` | Left-to-right | Arrow/Category composition: `filter even >>> map (*2) >>> sum` |
| `(<$>)` | — | Mapping inside a functor: `(+1) <$> Just 5` |
| `(<*>)` | — | Applying functions inside a functor: `User <$> parseName <*> parseAge` |
| `(>>=)` | Left-to-right | Dependent monadic chaining: `getLine >>= putStrLn` |
| `(>>)` | Left-to-right | Sequencing, discarding first result: `putStr "Name: " >> getLine` |
| `(<>)` | — | Semigroup append: `"Hello" <> " " <> "World"` |

Several utility combinators appear constantly:

```haskell
-- on (Data.Function): project before comparing
sortBy (compare `on` length) xs       -- sort by length
groupBy ((==) `on` fst) pairs         -- group by first element

-- bool (Data.Bool): point-free conditional
classify = bool "odd" "even" . even    -- bool falseCase trueCase predicate

-- id: identity function, useful as default/no-op
applyIf True f = f
applyIf False _ = id

-- const: ignore second argument
map (const 0) [1,2,3]                 -- [0,0,0]
```

Prefer `(<>)` over `(++)` for polymorphic code — it works on any `Semigroup`, not just lists. Prefer `comparing` from `Data.Ord` over `compare `on``:

```haskell
sortBy (comparing length) xs           -- ascending by length
sortBy (comparing Down . length) xs    -- descending
```

---

## Do-notation: when to use it and when to drop it

**Single-statement do blocks are always redundant** — just remove `do`:

```haskell
-- ❌ greet name = do putStrLn ("Hello " ++ name)
-- ✅ greet name = putStrLn ("Hello " ++ name)
```

**`x <- action; return x`** simplifies to just `action`:

```haskell
-- ❌ getName = do { x <- getLine; return x }
-- ✅ getName = getLine
```

**Independent bindings** should use Applicative style, not do-notation:

```haskell
-- ❌                              -- ✅
do a <- getA                       (,) <$> getA <*> getB
   b <- getB                       -- or: liftA2 (,) getA getB
   return (a, b)
```

**Simple chains** work well with `(>>=)`:

```haskell
getLine >>= putStrLn . ("Hello, " ++)
lookup key m >>= parseValue >>= validate     -- Maybe chaining
```

**Kleisli composition** `(>=>)` creates point-free monadic pipelines:

```haskell
processAndSave = parseInput >=> validate >=> save
-- equivalent to: \x -> parseInput x >>= validate >>= save
```

Reserve `do` for **complex multi-step computations** with many intermediate bindings, control flow, or `let` bindings — where operator chaining would be harder to follow. Inside do-blocks, use `let` for pure bindings and `<-` for monadic actions. Modern Haskell **prefers `pure` over `return`** — it requires only `Applicative`, the weaker constraint.

---

## Pattern matching made terse with GHC extensions

**LambdaCase** (`\case`, included in GHC2021) eliminates throwaway variables:

```haskell
-- ❌ eitherToMaybe e = case e of { Left _ -> Nothing; Right x -> Just x }
-- ✅ eitherToMaybe = \case { Left _ -> Nothing; Right x -> Just x }

-- Especially powerful with >>= :
peekWord8' >>= \case
    0xF1 -> mtcQuarter
    0xF2 -> songPosition
    _    -> empty
```

GHC 9.4+ adds **`\cases`** for multi-argument lambda case. **MultiWayIf** provides guard-style syntax in expression position, eliminating nested `if-else-if` chains:

```haskell
-- ❌ Nested if-else-if — hard to read, error-prone indentation
if has2 && has3 && has1 then ContentConflict path
else if has2 && has3 && not has1 then AddAdd path
else if has2 && not has3 then ModifyDelete path False
else ContentConflict path

-- ✅ MultiWayIf — flat, aligned, easy to extend
if | has2 && has3 && has1     -> ContentConflict path
   | has2 && has3 && not has1 -> AddAdd path
   | has2 && not has3         -> ModifyDelete path False
   | has3 && not has2         -> ModifyDelete path True
   | otherwise                -> ContentConflict path
```

MultiWayIf is especially useful inside `do` blocks, lambdas, and `foldl'` accumulators where top-level guards aren't available:

```haskell
-- Inside a fold accumulator
foldl' (\(afterDash, acc) arg ->
    if | arg == "--" -> (True, acc)
       | afterDash   -> (True, arg:acc)
       | isFlag arg  -> (False, acc)
       | otherwise   -> (False, arg:acc)
    ) (False, []) args
```

It also works well with `pure` to factor out the monadic wrapper:

```haskell
-- ❌ if cond1 then pure result1 else if cond2 then pure result2 else pure result3
-- ✅ pure $ if | cond1 -> result1 | cond2 -> result2 | otherwise -> result3
```

**Pattern guards** allow failable pattern matches inside guards:

```haskell
addLookup env var1 var2
  | Just val1 <- lookup env var1
  , Just val2 <- lookup env var2
  = Just (val1 + val2)
addLookup _ _ _ = Nothing
```

**View patterns** (`ViewPatterns`, GHC2021) apply a function and match on the result:

```haskell
size (view -> Unit)        = 1
size (view -> Arrow t1 t2) = size t1 + size t2
```

**Use eliminators** (`maybe`, `either`, `bool`) instead of explicit case when the function is a simple transform:

```haskell
-- ❌ case mx of { Nothing -> 0; Just x -> x }
-- ✅ fromMaybe 0 mx

-- ❌ case result of { Left e -> handleErr e; Right v -> process v }
-- ✅ either handleErr process result
```

---

## Functor, Applicative, and Monad: choosing the right level

The fundamental rule: **use the weakest abstraction that works**. If effects are independent, use Applicative. If a later action depends on an earlier result, use Monad.

```haskell
-- Applicative: all fields parsed independently (enables error accumulation)
User <$> parseName <*> parseAge <*> parseEmail

-- Monadic: second action depends on first
do name <- getLine
   if null name then fail "empty" else lookupUser name
```

Key traverse/map/sequence patterns:

```haskell
traverse validate fields              -- Applicative traversal (preferred in general)
mapM processItem items                -- Monadic traversal (fine in IO context)
for_ items $ \item -> do              -- "for loop" style when action is multi-line
  process item; log item
mapM_ print items                     -- short one-liner: mapM_ reads naturally
sequence [getLine, getLine, getLine]  -- [IO String] -> IO [String]
```

**`for_`/`forM_`** (flipped argument order) reads like an imperative loop and is preferred when the action body is large. **`traverse_`/`mapM_`** is preferred when the function is already named.

Other essential combinators:

```haskell
when debug $ putStrLn "debug mode"           -- conditional action
unless (null errors) $ mapM_ report errors   -- negated conditional
void $ forkIO someAction                     -- discard return value
guard (age >= 18) $> ticket                  -- Maybe/list filter
join (Just (Just 3))                         -- Just 3 (flatten nested monad)
getUserName userId <&> Text.toUpper          -- <&> is flipped <$>, pipeline style
```

**`traverse_` is the idiomatic way to perform an action on a `Maybe` value**, replacing the verbose `maybe (pure ()) action` pattern:

```haskell
-- ❌ maybe (pure ()) killThread mThread
-- ✅ traverse_ killThread mThread

-- ❌ case mHandle of Just h -> hClose h; Nothing -> pure ()
-- ✅ traverse_ hClose mHandle
```

**`void` replaces `_ <- action`** when discarding a monadic result:

```haskell
-- ❌ _ <- scannedEnv
-- ✅ void scannedEnv

-- ❌ _ <- Git.runGitRaw ["commit", "-m", msg]
-- ✅ void $ Git.runGitRaw ["commit", "-m", msg]
```

**`const` replaces `\_ -> expr`** for ignored lambda arguments:

```haskell
-- ❌ bracket acquire (\_ -> cleanup) action
-- ✅ bracket acquire (const cleanup) action

-- ❌ either (\_ -> handleErr) process result
-- ✅ either (const handleErr) process result
```

**`isRight`/`isLeft`** from `Data.Either` replace `either (const False) (const True)`:

```haskell
-- ❌ either (const False) (const True) (decodeUtf8' bs)
-- ✅ isRight (decodeUtf8' bs)
```

**`foldM`** is the monadic fold, **`filterM`** the monadic filter, and **`concatMapM f = fmap concat . mapM f`** (not in base but trivially composed).

---

## Lens and optics for nested data manipulation

The `lens` library's three core operations — **view** (`^.`), **set** (`.~`), **over** (`%~`) — compose with ordinary `(.)` and chain with `(&)`:

```haskell
person ^. address . city                     -- nested get
person & address . city .~ "NYC"             -- nested set
person & age %~ (+1)                         -- modify through function
molecule & atoms . traversed . point . x %~ (+1)  -- map over all atoms' x coords
```

Essential optics for collections:

```haskell
m ^. at "alice"                    -- Maybe value at key (can insert/delete)
m & at "key" .~ Just 42           -- insert
m & at "key" .~ Nothing           -- delete
m & ix "key" %~ (+1)              -- modify if present, no-op otherwise
[1..10] ^.. traversed . filtered even        -- [2,4,6,8,10]
(1,2) & both %~ (*10)                        -- (10, 20)
Just 5 & _Just %~ (+1)                       -- Just 6
```

**Stateful operations** (in `MonadState`) use `=` suffixed operators: `.=` for set, `%=` for modify, `+=` for increment. Use `use` to read from state:

```haskell
entities . ix 0 . position . x += 1         -- in a State/StateT context
score <- use (entities . to length)
```

Generate lenses with **`makeLenses`** (prefix fields with `_`), **`makeClassy`** (generates `HasFoo` typeclass for embedding), or **`makeFields`** (per-field typeclasses for overloaded access). Lens shines for **deeply nested updates and traversals**; for single-level record access, plain pattern matching is clearer.

The **optics** library is an alternative with clearer type errors and an opaque `Optic` newtype (uses `%` for composition instead of `.`), but `lens` remains the de facto standard on Hackage.

---

## Records, newtypes, and deriving strategies

**Modern record extensions** eliminate most boilerplate. The recommended combo for GHC 9.2+:

```haskell
{-# LANGUAGE NoFieldSelectors, OverloadedRecordDot, DuplicateRecordFields #-}

data Person = Person { name :: String, age :: Int }
data Company = Company { name :: String, owner :: Person }

main = print $ company.owner.name        -- OOP-style dot access
names = map (.name) people               -- section syntax
```

**RecordWildCards** and **NamedFieldPuns** enable concise construction and destructuring:

```haskell
greet User{..} = "Hello, " <> name                -- RecordWildCards: all fields in scope
greet User{name, age} = name <> " (" <> show age <> ")"  -- NamedFieldPuns: explicit subset
mkUser name age = User{..}                          -- construction from matching bindings
```

**DerivingStrategies** (GHC 8.2+) gives explicit control. Always specify the strategy:

```haskell
newtype Name = Name Text
  deriving stock    (Show, Generic)          -- compiler-generated
  deriving newtype  (Eq, IsString)           -- through the wrapped type (GND)
  deriving anyclass (ToJSON, FromJSON)        -- default methods (typically Generic-based)
```

**DerivingVia** (GHC 8.6+) enables deriving through *any* type with the same runtime representation:

```haskell
newtype Score = Score Int
  deriving (Semigroup, Monoid) via (Sum Int)   -- monoidal addition

data FilmRating = U | PG | R
  deriving stock (Bounded, Eq, Ord)
  deriving (Semigroup, Monoid) via (Max FilmRating)  -- "most restrictive" monoid
```

Common DerivingVia targets include **`Sum`**, **`Product`**, **`Ap`/`App`** (lift Monoid through Applicative), **`Alt`** (Monoid from Alternative), and **`Endo`** (composition monoid). Newtypes are **zero-cost at runtime** and should be used liberally for type safety:

```haskell
newtype UserId    = UserId Int    deriving newtype (Eq, Ord, Show)
newtype ProductId = ProductId Int deriving newtype (Eq, Ord, Show)
-- getUser (ProductId 5) is now a type error, not a silent bug
```

---

## Strings, text, and the five string types

| Type | Backed by | Use for |
|---|---|---|
| `String` (`[Char]`) | Linked list | Show/Read, compatibility only |
| `Data.Text` (strict) | UTF-8 array | **Default for all text** |
| `Data.Text.Lazy` | Chunks | Streaming, builders |
| `Data.ByteString` (strict) | Byte array | Binary data, network I/O |
| `Data.ByteString.Lazy` | Chunks | Large binary, streaming |

**Always enable `OverloadedStrings`** so string literals work polymorphically. Convert between types explicitly:

```haskell
T.pack / T.unpack                   -- String ↔ Text
TE.encodeUtf8 / TE.decodeUtf8'     -- Text ↔ ByteString (UTF-8, safe version)
TL.toStrict / TL.fromStrict        -- Lazy ↔ Strict Text
```

For efficient text construction, use **`Data.ByteString.Builder`** or **`Data.Text.Lazy.Builder`** rather than repeated `(<>)`. Libraries like **PyF** (`[fmt|Hello {name}|]`), **string-interpolate** (`[i|Hello #{name}|]`), and **fmt** (`"Hello "+|name|+""`) provide string interpolation.

---

## Lists, maps, and collection patterns

**Prefer higher-order functions over explicit recursion.** The choice of fold matters:

- **`foldr`**: building data structures, short-circuiting (e.g., `any`, `all`), infinite lists
- **`foldl'`** (strict): reducing to a value (sums, counts). **Never use lazy `foldl`** — it builds thunks and causes space leaks
- **`foldMap`**: when elements map into a Monoid

Essential one-liners:

```haskell
mapMaybe readMaybe ["1","x","3"] :: [Int]    -- [1, 3] — map + filter in one pass
catMaybes [Just 1, Nothing, Just 3]          -- [1, 3]
partitionEithers results                      -- ([errors], [successes])
concatMap expand items                        -- flatMap equivalent
sortOn length xs                              -- Schwartzian transform, O(n log n)
```

**`NonEmpty`** from `Data.List.NonEmpty` guarantees non-emptiness at the type level — `NE.head` is total. Use it wherever an empty collection is invalid.

**Map idioms** center on `fromListWith`, `insertWith`, `unionWith` for combining duplicate keys, and the **two-line import pattern**:

```haskell
import Data.Map.Strict (Map)
import qualified Data.Map.Strict as Map
```

Avoid `nub` on large lists (**O(n²)**) — use `Set.toList . Set.fromList` or `nubOrd` from the `extra` package.

---

## Error handling: the hierarchy from Maybe to exceptions

| Approach | When to use |
|---|---|
| `Maybe` | Simple absence, no error info needed |
| `Either e` | Expected failures with structured error types (parsers, validation) |
| `ExceptT e m` (non-IO base) | Threading typed errors through pure transformer stacks |
| IO exceptions (`throwIO`/`catch`) | Unexpected IO failures, resource cleanup |

**Do not wrap `ExceptT` around IO** — this is a well-known anti-pattern (documented by Michael Snoyman). It creates a third error channel on top of synchronous and asynchronous IO exceptions.

The **`note`** pattern converts Maybe to Either: `note "not found" (lookup key m)`. The **`safe`** library provides total versions of partial functions: `headMay`, `readMay`, `atMay`. Use `bracket` for exception-safe resource management and `safe-exceptions` for production code that properly distinguishes sync vs async exceptions.

```haskell
-- Pure code: return Either
parseConfig :: String -> Either ConfigError Config

-- IO code: throw exceptions, catch at boundaries
loadConfig :: FilePath -> IO Config
loadConfig path = do
  contents <- readFile path
  either (throwIO . ConfigException) pure (parseConfig contents)
```

---

## Type-level idioms that make code more concise

**TypeApplications** (GHC2021) eliminates the need for type annotations and `Proxy`:

```haskell
read @Int "42"               -- instead of (read "42" :: Int)
maxBound @Int                -- instead of maxBound :: Int
show @Int                    -- monomorphic show
```

**ScopedTypeVariables** (GHC2021, requires explicit `forall`) lets type variables scope into `where` clauses:

```haskell
f :: forall a. [a] -> [a]
f xs = ys ++ ys
  where ys :: [a]        -- same 'a' as the top-level signature
        ys = reverse xs
```

**RankNTypes** (GHC2021) enables higher-rank polymorphism — the argument itself must be polymorphic:

```haskell
runST :: (forall s. ST s a) -> a       -- s cannot escape, enforcing purity
type Lens s t a b = forall f. Functor f => (a -> f b) -> s -> f t
```

---

## GHC extensions every AI assistant should enable

The most impactful extensions for terse, modern Haskell:

- **`OverloadedStrings`**: string literals work as `Text`, `ByteString`, etc.
- **`LambdaCase`**: `\case` eliminates throwaway pattern match variables
- **`MultiWayIf`**: flat guard syntax in expression position, replacing nested `if-else-if`
- **`BlockArguments`** (GHC2024): `when condition do ...` without `$`
- **`TupleSections`**: `(,3)` instead of `\x -> (x, 3)`
- **`TypeApplications`**: `read @Int` instead of type annotations
- **`ScopedTypeVariables`**: type variables scope into where clauses
- **`DerivingStrategies`**: explicit `stock`/`newtype`/`anyclass`/`via`
- **`RecordWildCards`**: `MyRecord{..}` for destruction and construction
- **`OverloadedRecordDot`** (GHC 9.2+): `record.field` syntax
- **`NumericUnderscores`**: `1_000_000` for readability
- **`ApplicativeDo`**: desugars independent do-bindings to Applicative

The **GHC2021** language edition bundles most of these. **GHC2024** adds `BlockArguments`.

---

## Import patterns and module organization

Follow the **two-line import pattern** for containers and text:

```haskell
import Data.Map.Strict (Map)
import qualified Data.Map.Strict as Map

import Data.Text (Text)
import qualified Data.Text as T           -- or 'Text' per Kowainik

import qualified Data.ByteString as BS
import qualified Data.Set as Set
import qualified Data.List.NonEmpty as NE
```

Group imports: (1) unqualified external, (2) unqualified internal, (3) qualified external, (4) qualified internal, separated by blank lines. Use **explicit import lists** for small imports and **qualified imports** when a module is designed for it (containers, text, bytestring). Consider a **custom prelude** like `relude` (no partial functions, `Text` as default string type) or `protolude` for production projects. The **`ImportQualifiedPost`** extension allows `import Data.Map qualified as Map` for cleaner alignment.

---

## Where vs let: scoping auxiliary definitions

**Use `where`** for function-level helpers that support guards and keep main logic first:

```haskell
bmiTell weight height
  | bmi <= 18.5 = "Underweight"
  | bmi <= 25.0 = "Normal"
  | otherwise   = "Overweight"
  where bmi = weight / height ^ 2
```

**Use `let`** for intermediate pure bindings in do-blocks:

```haskell
main = do
  input <- getLine
  let parsed = read @Int input
  print (parsed * 2)
```

Avoid deeply nesting `where` inside `where` — extract helpers to the top level instead.

---

## Consolidation and DRY patterns

**Consolidate multiple `liftIO` calls** into a single `liftIO $ do` block when consecutive actions all need lifting:

```haskell
-- ❌ Repetitive lifting
liftIO $ putStrLn "Updating..."
liftIO $ void $ Git.runGitRaw ["commit", "-m", msg]
liftIO $ putStrLn "Done."

-- ✅ Single lift, cleaner do-block
liftIO $ do
    putStrLn "Updating..."
    void $ Git.runGitRaw ["commit", "-m", msg]
    putStrLn "Done."
```

**Extract repeated lambdas into named helpers** when the same inline function appears multiple times:

```haskell
-- ❌ Same truncation lambda repeated 4 times across the module
mapM_ (printVerifyIssue (\s -> take 16 s ++ if length s > 16 then "..." else "")) issues

-- ✅ Named helper, used everywhere
truncateHash :: String -> String
truncateHash s = take 16 s ++ if length s > 16 then "..." else ""

mapM_ (printVerifyIssue truncateHash) issues
```

**Reuse utility functions instead of inlining the same logic** across files:

```haskell
-- ❌ Same lambda duplicated in 3 files
let safePath = map (\c -> if c == '\\' then '/' else c) absIndex

-- ✅ Import from the utility module
import Bit.Utils (toPosix)
let safePath = toPosix absIndex
```

**`dropWhileEnd` replaces `reverse . dropWhile p . reverse`** for trimming trailing elements:

```haskell
-- ❌ Reverse-strip-reverse idiom
trim = reverse . dropWhile isSpace . reverse . dropWhile isSpace

-- ✅ dropWhileEnd (Data.List) — clearer, single pass on the end
trim = dropWhileEnd isSpace . dropWhile isSpace
```

---

## The anti-pattern catalogue: verbose patterns to replace

These are the most common mistakes that beginners and AI models produce. Each should be recognized and rewritten:

**Explicit recursion instead of standard combinators:**
```haskell
-- ❌ sumList [] = 0; sumList (x:xs) = x + sumList xs
-- ✅ sumList = foldl' (+) 0
```

**Unnecessary do-notation for pure code:**
```haskell
-- ❌ greet name = do putStrLn ("Hello " ++ name)
-- ✅ greet name = putStrLn ("Hello " ++ name)
```

**Pattern matching on Maybe/Either just to re-wrap:**
```haskell
-- ❌ case mx of { Nothing -> Nothing; Just x -> Just (x+1) }
-- ✅ fmap (+1) mx
```

**`length xs == 0` instead of `null xs`** — `null` is O(1), `length` is O(n).

**`concat . map f` instead of `concatMap f`** — a single, clearer function.

**Nested if-then-else instead of guards:**
```haskell
-- ❌ if n < 0 then "neg" else if n == 0 then "zero" else "pos"
-- ✅ | n < 0 = "neg" | n == 0 = "zero" | otherwise = "pos"
```

**Lazy `foldl` instead of strict `foldl'`** — causes space leaks on large inputs.

**`String` instead of `Text`** in production code — **40+ bytes per character** vs packed UTF-8.

**Using `head`/`tail`/`fromJust`/`read`** — partial functions that crash. Use `listToMaybe`, pattern matching, `readMaybe`, and `fromMaybe`.

**`maybe (pure ()) action value` instead of `traverse_ action value`** — the `Foldable` method is exactly this pattern:
```haskell
-- ❌ maybe (pure ()) killThread mThread
-- ✅ traverse_ killThread mThread
```

**`_ <- action` instead of `void action`** — `void` from `Control.Monad` is the standard combinator for discarding results.

**`\_ -> expr` instead of `const expr`** — `const` is point-free and self-documenting.

**`if code == ExitSuccess then ... else ...` instead of `case code of`** — treats a sum type as a boolean; pattern matching is more expressive and enables exhaustiveness checking.

**`either (const False) (const True)` instead of `isRight`** — the `Data.Either` module exports exactly this predicate.

**`maybe [] (:[]) (listToMaybe xs)` instead of `take 1 xs`** — recognizing round-trip compositions that cancel out.

**Orphan instances** — break global instance uniqueness. Use newtype wrappers.

**Lazy IO (`readFile`)** — unpredictable resource management. Use strict `Data.Text.IO.readFile` or streaming libraries.

**The existential typeclass anti-pattern** — wrapping values in `forall a. Show a => AnyShow a` when `[String]` suffices.

---

## Conclusion

Idiomatic Haskell achieves concision not through clever tricks but through **choosing the right abstraction level**: `fmap` over pattern matching, Applicative over Monad, `foldl'` over explicit recursion, eliminators (`maybe`, `either`, `bool`) over `case`, and composition over intermediate variable naming. The critical meta-principle is to **use the weakest sufficient tool** — this produces code that is both shorter and more general.

Three practices have the single highest impact on code quality: enabling `OverloadedStrings` and using `Text` by default, compiling with `-Wall -Werror`, and using `DerivingStrategies` to make instance derivation explicit. An AI assistant that consistently applies the patterns in this reference — from `\case` and `MultiWayIf` to `traverse_`/`void`/`const`, from `<$>`/`<*>` chains to `dropWhileEnd` and `isRight` — will produce Haskell that experienced developers recognize as fluent rather than translated from another language.
```

---

## docs/remote-commands-tutorial.md

**Path:** `docs/remote-commands-tutorial.md`

*Source file.*

```markdown
# Remote Commands Tutorial

## The Problem

You have 10 GB of video footage already sitting on Google Drive. You want bit to
track it, but downloading everything locally just to run `bit add` and then
uploading it all back is a waste of time and bandwidth. The files are already
where they need to be.

Remote commands let you build metadata *for files already on the remote* without
downloading them. bit reads hashes directly from the cloud, downloads only small
text files for classification, and never touches the large binaries.

## Syntax

Two equivalent forms:

```
bit --remote <name> <command>     # portable, works everywhere
bit @<name> <command>             # shorthand, needs quoting in PowerShell
```

Use `--remote` in scripts. Use `@` interactively if your shell supports it.

In PowerShell, `@` is the splatting operator, so `@origin` is silently eaten.
Either quote it (`'@origin'`) or use `--remote origin` instead.

## Quick Start

```bash
# 1. Create a local bit repo (no files needed locally)
mkdir my-project && cd my-project
bit init

# 2. Point it at your remote
bit remote add origin gdrive:Projects/footage

# 3. Initialize tracking on the remote
bit --remote origin init

# 4. Scan the remote and commit metadata
bit --remote origin add .

# 5. Pull metadata locally (instant — just the bundle, not the files)
bit pull
```

After step 4, the remote has a `.bit/bit.bundle` containing full Git history of
every file's hash and size. After step 5, your local repo knows about all 847
files even though your working directory is still empty. From here you can
`bit push`, `bit pull`, or inspect history with `bit log`.

## Commands

### `init` — Create an empty repository on the remote

```bash
bit --remote origin init
```

This creates an empty Git bundle on the remote at `.bit/bit.bundle`. It does
**not** scan files — it only establishes the history so that subsequent commands
have something to work with.

Running it twice errors out:

```
fatal: remote already has a bit repository.
```

### `add` — Scan, classify, and commit metadata

```bash
bit --remote origin add .
bit --remote origin add data/
bit --remote origin add models/weights.bin
```

This is the workhorse command. On every call it:

1. Fetches the bundle from the remote
2. Inflates it into a temporary workspace
3. Scans the remote via `rclone lsjson --hash --recursive`
4. Classifies each file as text or binary:
   - Files above 1 MB or with known binary extensions (.mp4, .zip, etc.) are
     binary — no download needed
   - Small files are downloaded to a temp directory and inspected
5. Writes metadata into the workspace:
   - Binary files get a 2-line `hash:`/`size:` stub
   - Text files get their actual content
6. Stages the specified paths (or `.` for everything)
7. Auto-commits if there are changes
8. Re-bundles and pushes back to the remote

If nothing changed since the last add:

```
Nothing to add — remote metadata is up to date.
```

Example output for a fresh remote:

```
Scanning remote...
Found 12 files on remote.
  8 binary files (by size/extension)
  4 small files to classify...
Downloading 3 text files...
Remote metadata updated.
Creating metadata bundle...
Pushing bundle to remote...
Changes pushed to remote.
```

For a typical 10 GB media repo, this downloads maybe 50 KB of text files while
skipping the 10 GB of video.

### `status` — Check the workspace state

```bash
bit --remote origin status
```

Read-only. Fetches the bundle, inflates it, runs `git status`, and cleans up.
Nothing is pushed back. After a successful `add`, this shows:

```
On branch main
nothing to commit, working tree clean
```

All flags are passed through to git, so `bit --remote origin status --short`
works too.

### `log` — View commit history

```bash
bit --remote origin log
bit --remote origin log --oneline
bit --remote origin log --oneline -5
```

Read-only. Shows the Git history of the remote metadata. Example:

```
a3f1b2c Update remote metadata
96c9f37 Initial remote repository
```

### `commit` — Amend or extend commits

```bash
bit --remote origin commit --amend -m "Initial metadata scan"
```

This is for editing the history after `add` has auto-committed. Common uses:

- Amend the last commit message
- Create additional commits after manual workspace modifications

The command fetches the bundle, inflates it, runs `git commit` with your args,
re-bundles, and pushes. If the commit fails (e.g., nothing to commit), the
bundle is not pushed.

## Complete Walkthrough

### Scenario: Media production archive on Google Drive

You have a Google Drive folder `gdrive:Archive/2025-shoot` containing:

```
2025-shoot/
├── raw/
│   ├── take01.mov      (2.1 GB)
│   ├── take02.mov      (1.8 GB)
│   └── take03.mov      (3.4 GB)
├── audio/
│   ├── ambient.wav     (450 MB)
│   └── interview.wav   (820 MB)
├── notes.txt           (2 KB)
├── shot-list.md        (5 KB)
└── README.md           (1 KB)
```

#### Step 1: Set up local repo

```bash
mkdir shoot-tracker && cd shoot-tracker
bit init
bit remote add origin gdrive:Archive/2025-shoot
```

#### Step 2: Initialize the remote

```bash
$ bit --remote origin init
Initialized bit repository on remote 'origin'.
```

#### Step 3: Scan and commit metadata

```bash
$ bit --remote origin add .
Scanning remote...
Found 8 files on remote.
  5 binary files (by size/extension)
  3 small files to classify...
Downloading 3 text files...
Remote metadata updated.
Creating metadata bundle...
Pushing bundle to remote...
Changes pushed to remote.
```

bit scanned all 8 files. The 5 large media files (.mov, .wav) were classified
as binary by extension — their MD5 hashes came directly from Google Drive's
metadata API via `rclone lsjson --hash`, no download required. The 3 small text
files (notes.txt, shot-list.md, README.md) were downloaded, classified, and
their content stored directly in Git.

Total data downloaded: ~8 KB (the three text files).
Total data **not** downloaded: ~8.57 GB (the five media files).

#### Step 4: Give it a better commit message

```bash
$ bit --remote origin commit --amend -m "Archive 2025 shoot: 5 media + 3 text"
[main 7e2a1f3] Archive 2025 shoot: 5 media + 3 text
 Date: Mon Feb 9 12:00:00 2026 +0000
 8 files changed, 13 insertions(+)
Creating metadata bundle...
Pushing bundle to remote...
Changes pushed to remote.
```

#### Step 5: Verify and pull locally

```bash
$ bit --remote origin log --oneline
7e2a1f3 Archive 2025 shoot: 5 media + 3 text
96c9f37 Initial remote repository

$ bit pull
```

Now your local bit repo has the full metadata history. `bit log` shows the same
commits. `bit status` knows about all 8 files. If you later want the actual
media files locally, `bit pull` will download them.

### Scenario: Filesystem remote (USB drive)

Remote commands work with filesystem remotes too, though they are most useful
with cloud storage where you want to avoid unnecessary transfers.

```bash
bit init
bit remote add usb E:\backups\project
bit --remote usb init
bit --remote usb add .
```

### Scenario: Re-scanning after remote changes

If files change on the remote (someone uploads new footage), run `add` again:

```bash
$ bit --remote origin add .
Scanning remote...
Found 10 files on remote.
  7 binary files (by size/extension)
  3 small files to classify...
Downloading 3 text files...
Remote metadata updated.
Creating metadata bundle...
Pushing bundle to remote...
Changes pushed to remote.
```

Each `add` does a full re-scan. The workspace is ephemeral — it starts from
the bundle every time and reconstructs the complete metadata from the current
remote state. Additions, modifications, and deletions are all detected
automatically.

## How It Works Under the Hood

Every remote command follows the same pattern:

```
fetch bundle from remote
  → inflate into system temp directory
    → operate (scan, add, commit, status, log)
    → re-bundle if changes were made
  → push new bundle to remote
→ clean up temp directory
```

There is no persistent workspace on disk between commands. The remote's
`.bit/bit.bundle` is the sole source of truth. Temp directories are cleaned up
even if the command fails (exception-safe via `bracket`).

Read-write commands (`add`, `commit`) compare Git HEAD before and after the
action. If HEAD didn't change, the push is skipped — no unnecessary uploads.

Read-only commands (`status`, `log`) never push, even if they technically could.

## Error Messages

| Situation | Message |
|-----------|---------|
| Remote not initialized | `fatal: no bit repository on remote. Run 'bit @remote init' first.` |
| Already initialized | `fatal: remote already has a bit repository.` |
| Remote not configured | `fatal: remote 'foo' not found.` |
| Not in a bit repo | `fatal: not a bit repository (or any of the parent directories): .bit` |
| Network failure | `fatal: network error: <details>` |
| Unsupported command | `error: command not supported in remote context: push` |

## Tips

- **`init` then `add`**: Always run `init` before `add`. `init` creates the
  empty history; `add` populates it. They are separate because `init` is a
  one-time setup, while `add` can be run repeatedly as files change.

- **Amend freely**: The auto-commit message from `add` is generic ("Update
  remote metadata"). Use `commit --amend -m "..."` to give it a meaningful name
  before pulling locally.

- **Check before pulling**: Use `log --oneline` and `status` to inspect the
  remote state before pulling metadata into your local repo.

- **Cloud remotes benefit most**: For filesystem remotes, you could just `cd`
  to the remote and run bit commands there directly. Remote commands shine when
  the remote is a cloud backend where direct access isn't possible.

- **Text classification is conservative**: Files that can't be downloaded for
  classification are treated as binary. This is safe — binary metadata is
  always correct, while misclassifying a binary as text would store garbage in
  Git.
```

---

## docs/spec.md

**Path:** `docs/spec.md`

*Source file.*

```markdown
# bit Implementation Specification (v3)

## Context and Vision

**bit** is a version control system designed for large files that leverages Git as a metadata-tracking engine while storing actual file content separately. The core insight: Git excels at tracking small text files, so we feed it exactly that — tiny metadata files instead of large binaries.

**Mental Model**: bit = Git(metadata) + rclone(sync) + [optional CAS(content) for bit-solid]

### bit-lite vs Git vs bit-solid: Content Authority

The three systems differ in where content lives and what guarantees that provides:

**Git** stores file content directly in its object store (`.git/objects/`). Every blob, tree, and commit is content-addressed by SHA-1. When you push, you're transferring objects that are self-verifying. When you pull, the objects you receive are self-verifying. Metadata (commits, trees) and content (blobs) live in the same store. The object store is the single source of truth, and it can always back up any metadata claim.

**bit-solid** (planned) will add a content-addressed store (CAS) alongside Git's metadata tracking. Like Git, every version of every file will be stored by its hash. The CAS backs up every metadata claim unconditionally — if the metadata says a file existed at commit N with hash X, the CAS has that blob. This enables full binary history, time travel, and sparse checkout.

**bit-lite** (current) has no object store and no CAS. Git tracks only metadata files (2-line hash+size records). The actual binary content lives exclusively in the working tree. There is exactly one copy of each file — the one on disk right now. Old versions are gone the moment you overwrite a file.

This creates a fundamental architectural constraint that Git and bit-solid don't have: **metadata can become hollow.** If a binary file is deleted, corrupted, or modified without running `bit add`, the metadata in `.bit/index/` claims something that is no longer true. In Git, this situation is impossible — the object store is append-only and self-verifying. In bit-lite, it's the normal consequence of working with mutable files on a regular filesystem.

This constraint gives rise to the **proof of possession** rule (see below): bit-lite must verify that content matches metadata before transferring metadata to or from another repo. Without this rule, hollow metadata propagates between repos, and the system's core value proposition — knowing the true state of your files — is undermined.

### Origin

The key architectural idea: instead of a custom manifest, keep small text files in `.bit/index/` mirroring the working tree's directory structure, each containing just hash and size. Define Git's working tree as `.bit/index/`, and you get `add`, `commit`, `diff`, `log`, branching, and the entire Git command set for free. Git becomes the manifest manager, diff engine, and history store — without ever seeing a large file.

### Comparison with Alternatives

bit occupies a different niche than git-lfs and git-annex:

- **git-lfs**: Stores pointer files in Git, actual files on a server. Server-dependent, transparent but limited. Pragmatic hack.
- **git-annex**: Extremely powerful distributed system with pluggable remotes and policy-driven placement. Extremely complex.
- **bit**: Minimal, explicit, correctness-oriented. Git never touches large files. Dumb remotes via rclone. Filesystem-first.

bit's killer feature is **clarity** — users always know what state their files are in, what bit is about to do, and how to recover from errors.

---

## Core Architecture

### Directory Structure

```
project/
├── actual_files/           # User's working directory (large files live here)
│   ├── src/
│   │   └── video.mp4
│   └── data/
│       └── dataset.bin
│
└── .bit/
    ├── index/              # Metadata files live here (Git working tree)
    │   ├── .git/           # Git's internal directory
    │   ├── src/
    │   │   └── video.mp4   # Metadata file (NOT the actual video)
    │   └── data/
    │       └── dataset.bin # Metadata file (NOT the actual data)
    ├── remotes/            # Named remote configs (typed)
    │   └── origin          # Remote type + optional target
    ├── devices/            # Device identity files
    ├── target              # Legacy: single remote URL
    └── ignore              # Gitignore-style rules
```

### The Index Invariant

**Git is the sole authority over `.bit/index/`.** After any git operation that
changes HEAD (merge, checkout, commit), `.bit/index/` is correct by
definition. The only remaining work is mirroring those changes onto the actual
working directory (downloading binaries from remote, copying text files from
the index).

This invariant applies uniformly across all pull/merge paths:

- `git merge` → determines correct metadata → we sync actual files
- `git checkout` (first pull, `--accept-remote`) → determines correct metadata → we sync actual files

No path should write metadata files to `.bit/index/` directly and then commit.
Scanning the remote via rclone and writing the result to the index bypasses git
and **will** produce wrong metadata (rclone cannot distinguish text from binary,
so text files get `hash:/size:` metadata instead of their actual content).

### The Proof of Possession Rule

In Git and in bit-solid (planned), content is always recoverable from an object store. Git stores file content in `.git/objects/`; bit-solid will store it in a content-addressed store (CAS). In both systems, metadata and content are either the same thing or the content is always reachable from the metadata. You can push any metadata you want — the objects back it up unconditionally.

bit-lite is fundamentally different. There is no object store. The working tree is the **only copy** of binary file content. The metadata in `.bit/index/` is just a *claim* — "there exists a file at this path with this hash and this size." If that file is gone, corrupted, or modified since the last commit, the claim is hollow. Pushing hollow claims to a remote means the remote now has metadata that nobody can fulfill. Pulling hollow claims from a remote means your local repo has metadata pointing to content that doesn't exist.

This is the **proof of possession** rule: a repo must not transfer metadata it cannot back up with actual content.

```
             Git / bit-solid                    bit-lite
             ──────────────                     ────────
Content:     Object store (always available)    Working tree (only copy)
Metadata:    Always backed by objects           Only valid if working tree matches
Push safety: Unconditional                      Requires verification first
Pull safety: Unconditional                      Requires remote verification first
```

The rule applies symmetrically:

**Push (sender must prove possession):**
1. Verify local — every binary file's hash must match its metadata
2. If verification fails, refuse to push
3. If verified, push metadata then copy files

**Pull (sender must prove possession):**
1. Verify remote — every binary file on the remote must match the remote's metadata
2. If verification fails, refuse to pull (suggest `--accept-remote`, `--force`, or `--manual-merge`)
3. If verified, pull metadata then copy files

**What happens when verification fails:**

On push failure (local doesn't match metadata):
```
$ bit push
error: Working tree does not match metadata.
  Modified: data/model.bin (expected md5:a1b2..., got md5:f8e9...)
  Missing:  data/weights.bin
hint: Run 'bit verify' to see all mismatches.
hint: Run 'bit add' to update metadata, or 'bit restore' to restore files.
```

On pull failure (remote doesn't match its metadata):
```
$ bit pull
error: Remote files do not match remote metadata.
  Modified: data/model.bin (expected md5:a1b2..., got md5:c3d4...)
hint: Run 'bit verify --remote' to see all mismatches.
hint: Run 'bit pull --accept-remote' to accept the remote's actual file state.
hint: Run 'bit push --force' to overwrite remote with local state.
```

The existing divergence resolution mechanisms (`--accept-remote`, `--force`, `--manual-merge`) serve as the escape hatches when the proof of possession check fails.

**Why this matters:** Without this rule, corruption propagates silently through metadata. Repo A has a corrupted file, pushes metadata claiming the file is fine, Repo B pulls that metadata, and now both repos have a lie in their history. The proof of possession rule stops corruption at the boundary — you cannot export claims you can't substantiate.

**Cost:** Verification requires hashing every binary file, which means reading the entire working tree. For a repo with 100GB of binary files, this takes real time. This is the price of not having an object store — the working tree must be checked because it's the only source of truth. Future optimization: cache verification results keyed on (path, mtime, size) to skip re-hashing unchanged files.

**Remote verification cost by transport type:**

- **Cloud remotes (Google Drive, S3, etc.):** `rclone lsjson --hash` returns MD5 hashes as free metadata — Google Drive stores them natively. Verification is essentially a single API call followed by in-memory comparison. Cheap.
- **Cloud remotes (other backends):** Some backends provide native hashes, some don't. Where hashes aren't free, rclone may need to download file content to hash it. Check per-backend.
- **Filesystem remotes:** Requires reading and hashing every binary file on the remote. Same cost as local verification.

**Implementation status:** The proof of possession rule is fully implemented as of this version:

- `bit push` always verifies local working tree before pushing (unconditional — `--force` only affects ancestry check)
- `bit pull` verifies remote before pulling (unless `--accept-remote` or `--manual-merge`)
- `bit fetch` does NOT verify (fetch only transfers metadata, no file sync happens)
- `bit fetch` is silent when already up to date (no stdout), similar to `git fetch`
- Cloud remotes: verified via `Verify.verifyRemote` using `rclone lsjson --hash`
- Filesystem remotes: verified via `Verify.verifyLocalAt` which hashes the remote's working tree
- Verification runs in parallel using bounded concurrency (`Parallel 0` = auto-detect based on CPU cores)

### Metadata File Format

Each metadata file under `.bit/index/` mirrors the path of its corresponding real file and contains **ONLY**:

```
hash: md5:a1b2c3d4e5f6...
size: 1048576
```

Two fields only:
- `hash` — MD5 hash of file content (prefixed with `md5:`)
- `size` — File size in bytes

Parsing and serialization are handled by a single canonical module (`bit/Internal/Metadata.hs`) which enforces the round-trip property: `parseMetadata . serializeMetadata ≡ Just`.

**Known deviation from original spec**: The original spec called for SHA256 and `hash=`/`size=` (equals-sign) format. The implementation uses MD5 (matching rclone's native hash) and `hash: `/`size: ` (colon-space) format. This is intentional — MD5 is used everywhere for consistency with rclone comparisons. SHA256 may be added later alongside MD5, not as a replacement.

### Git Configuration

bit runs Git with:
- Git repository initialized in `.bit/index/.git` during `bit init`
- All git operations use the repo at `.bit/index/.git`
- Working tree is `.bit/index` (not the project root)
- `core.excludesFile` points to `.bit/ignore`

### File Handling

- Regular files (binary → metadata, text → content stored directly in index)
- Symlinks — **ignored** (too many edge cases with cloud remotes)
- Device files, sockets, named pipes — **ignored**
- Empty directories — **ignored** (not tracked; many cloud backends don't support them)

---

## Module Architecture

### Layer Contract

The codebase follows strict layer boundaries:

```
bit/Commands.hs → Bit.hs → Internal/Transport.hs → rclone (only here!)
                      ↓
                   Internal/Git.hs → git (only here!)
```

- **Internal/Transport.hs** — Dumb rclone wrapper. Knows how to `copyTo`, `moveTo`, `deleteFile`, `listJson`, `check`. Takes `Remote` + relative paths. Does NOT know about `.bit/`, bundles, `RemoteState`, or `FetchResult`. Captures rclone JSON output as raw UTF-8 bytes to correctly handle non-ASCII filenames (Hebrew, Chinese, emoji, etc.). Uses `bracket` for exception-safe subprocess resource cleanup.
- **Internal/Git.hs** — Dumb git wrapper. Knows how to run git commands. Takes args. Does NOT interpret results in domain terms.
- **Bit.hs** — Smart business logic. All domain knowledge lives here. Knows about remotes, bundles, `.bit/` layout, sync strategy. Calls Transport and Git, never calls `readProcessWithExitCode` directly.
- **bit/Commands.hs** — Entry point. Parses CLI, resolves the remote, builds `BitEnv`, dispatches to `Bit`.

### Key Types

| Type | Module | Purpose |
|------|--------|---------|
| `Hash (a :: HashAlgo)` | `bit.Types` | Phantom-typed hash — compiler distinguishes MD5 vs SHA256 |
| `Path` | `bit.Types` | Domain path (bit-tracked file path). `newtype` over `FilePath` to prevent transposition bugs. |
| `FileEntry` | `bit.Types` | Tracked `Path` + `EntryKind` (hash, size, `ContentType`) |
| `BitEnv` | `bit.Types` | Reader environment: cwd, local files, remote, force flags |
| `BitM` | `bit.Types` | `ReaderT BitEnv IO` — the application monad |
| `MetaContent` | `bit.Internal.Metadata` | Canonical metadata: hash + size, single parser/serializer |
| `Remote` | `bit.Remote` | Resolved remote: name + URL. Smart constructor, `remoteUrl` for Transport only |
| `RemoteState` | `bit.Remote` | Remote classification: Empty, ValidRgit, NonRgitOccupied, Corrupted, NetworkError |
| `FetchResult` | `bit.Remote` | Bundle fetch result: BundleFound, RemoteEmpty, NetworkError |
| `FetchOutcome` | `bit.Core.Fetch` | UpToDate, Updated { foOldHash, foNewHash }, FetchedFirst, FetchError |
| `GitDiff` | `bit.Diff` | Added, Modified, Deleted, Renamed — pure diff result |
| `RcloneAction` | `bit.Plan` | Copy, Move, Delete, Swap — concrete rclone operations |
| `FileIndex` | `bit.Diff` | Dual-indexed file map (byPath + byHash) for efficient diff/rename detection |
| `Resolution` | `bit.Conflict` | KeepLocal or TakeRemote — conflict resolution choice |
| `DeletedSide` | `Internal.Git` (re-exported by `bit.Conflict`) | DeletedInOurs or DeletedInTheirs — which side deleted in modify/delete conflict |
| `ConflictInfo` | `bit.Conflict` | ContentConflict, ModifyDelete path DeletedSide, AddAdd — conflict type from git ls-files -u |
| `NameStatusChange` | `Internal.Git` | Added, Deleted, Modified, Renamed, Copied — parsed `git diff --name-status` output (replaces bare `(Char, FilePath, Maybe FilePath)` tuple) |
| `DivergentFile` | `bit.Core.Pull` | Record for remote divergence: path, expected/actual hash and size (replaces bare 5-tuple) |
| `ScannedEntry` | `bit.Scan` | ScannedFile / ScannedDir — internal type for scan pass, replaces (FilePath, Bool) |
| `FileToSync` | `bit.Core.Transport` | TextToSync / BinaryToSync — text has no size, binary carries byte count |
| `BinaryFileMeta` | `bit.Verify` | Record: bfmPath, bfmHash, bfmSize — replaces bare (Path, Hash, Integer) tuple |
| `VerifyResult` | `bit.Verify` | Record: vrCount, vrIssues — replaces bare (Int, [VerifyIssue]) tuple |
| `VerifyTarget` | `bit.Core.Verify` | VerifyLocal or VerifyRemote — whether verify checks local or remote |
| `DeviceInfo` | `bit.Device` | UUID + storage type + optional hardware serial |
| `RemoteType` | `bit.Device` | RemoteFilesystem, RemoteDevice, RemoteCloud |

### Sync Pipeline

The sync pipeline is composed as pure function composition with effectful endpoints:

```
scan   :: FilePath → IO [FileEntry]        -- effectful (reads filesystem)
diff   :: FileIndex → FileIndex → [GitDiff]  -- pure!
plan   :: GitDiff → RcloneAction              -- pure!
exec   :: RcloneAction → IO ()               -- effectful (calls rclone)
```

The pure middle (`diff >>> plan`) is factored into `bit.Pipeline` and is fully property-testable:

```haskell
diffAndPlan :: [FileEntry] -> [FileEntry] -> [RcloneAction]  -- pure core
pushSyncFiles :: [FileEntry] -> [FileEntry] -> [RcloneAction]
pullSyncFiles :: [FileEntry] -> [FileEntry] -> [RcloneAction]
```

### Working Tree Sync: The `oldHead` Pattern

After any git operation that changes HEAD (merge, checkout), the working
directory must be updated to reflect what changed in `.bit/index/`.
The mechanism:

```haskell
oldHead <- getLocalHeadE                        -- 1. Capture HEAD *before* the git operation
-- ... git merge / git checkout ...             -- 2. Git changes HEAD + index
applyMergeToWorkingDir transport cwd oldHead    -- 3. Diff old HEAD vs new HEAD, sync files
```

`applyMergeToWorkingDir` uses `git diff --name-status oldHead newHead` to
determine what changed, then:
- **Added/Modified files**: download binary from remote, or copy text from index
- **Deleted files**: remove from working directory
- **Renamed files**: delete old, download/copy new

**CRITICAL**: `applyMergeToWorkingDir` always reads the actual HEAD after merge
from git (via `getLocalHeadE`). It never accepts `newHead` as a parameter. This
prevents the bug where `remoteHash` (the remote's tip) was passed instead of the
actual merged HEAD, causing local-only files to appear deleted during merge.

This is used consistently across:
- Clean merges (fast-forward and three-way)
- Conflict resolution merges
- `--accept-remote` (force-checkout)
- `mergeContinue`

The only exception is **first pull** (`oldHead = Nothing`): there is no
previous HEAD to diff against, so `transportSyncAllFiles` (full sync from
current HEAD after checkout) is used as fallback.

### File Transport Abstraction

To eliminate duplication between cloud and filesystem pull paths, the merge
orchestration logic is now unified with a `FileTransport` abstraction that
captures the differences in how files are copied:

```haskell
data FileTransport = FileTransport
  { transportDownloadFile :: FilePath -> FilePath -> SyncProgress -> IO ()
  , transportSyncAllFiles :: FilePath -> IO ()
  }
```

**Cloud transport** (rclone-based):
- `transportDownloadFile`: Downloads a single file via `rclone copyto`, or copies text from index
- `transportSyncAllFiles`: Scans remote via `rclone lsjson`, diffs against local, syncs files

**Filesystem transport** (direct file copy):
- `transportDownloadFile`: Copies a single file directly, or copies text from index
- `transportSyncAllFiles`: Uses `git ls-tree HEAD` after checkout, copies files from remote working tree

The unified `pullLogic` and `pullAcceptRemoteImpl` now accept a `FileTransport`
parameter. Both cloud and filesystem paths use the same merge orchestration,
`oldHead` capture pattern, conflict resolution (`Conflict.resolveAll`), and
tracking ref updates. The only difference is the transport used to copy files.

### Conflict Resolution

Conflict resolution is structured as a fold over a list of conflicts (`bit.Conflict`). Each conflict is resolved identically via `resolveConflict`, and the traversal guarantees every conflict is visited exactly once with correct progress tracking (1/N, 2/N, ...). The decision logic (KeepLocal vs TakeRemote) is cleanly separated from the git checkout/merge mechanics.

**Critical**: After resolving all conflicts, the merge commit must **always** be
created, regardless of whether the index has staged changes. When the user
chooses "keep local" (`--ours`), `git checkout --ours` + `git add` restores
HEAD's version — the index becomes identical to HEAD. A naïve `hasStagedChanges`
check would skip the commit, leaving `MERGE_HEAD` dangling. Git's `commit`
command always succeeds when `MERGE_HEAD` exists (it knows it's recording a
merge), even if the tree is identical to HEAD's tree. Skipping the commit
breaks the next push (ancestry check fails because HEAD was never advanced
past the merge).

---

## Command Line Interface (Git-Compatible)

**CRITICAL**: The CLI mirrors Git's interface. Users familiar with Git should feel immediately at home.

### Command Mapping

| Command | Git Equivalent | bit Behavior |
|---------|---------------|---------------|
| `bit init` | `git init` | Initialize `.bit/` with internal Git repo |
| `bit add <path>` | `git add` | Compute metadata, write to `.bit/index/`, stage in Git |
| `bit add .` | `git add .` | Add all modified/new files |
| `bit commit -m "msg"` | `git commit` | Commit staged metadata changes |
| `bit status` | `git status` | Show working tree vs metadata vs staged |
| `bit diff` | `git diff` | Show hash/size changes (human-readable) |
| `bit diff --staged` | `git diff --staged` | Show staged metadata changes |
| `bit log` | `git log` | Show commit history |
| `bit restore [options] [--] <path>` | `git restore` | Restore metadata; full git syntax: --staged, --worktree, --source=, etc. |
| `bit checkout [options] -- <path>` | `git checkout --` | Restore working tree from index (legacy syntax) |
| `bit reset` | `git reset` | Reset staging area |
| `bit rm <path>` | `git rm` | Remove file from tracking |
| `bit mv <src> <dst>` | `git mv` | Move/rename tracked file |
| `bit branch` | `git branch` | Branch management |
| `bit merge` | `git merge` | Merge branches |
| `bit remote add <name> <url>` | `git remote add` | Add named remote (does NOT set upstream) |
| `bit remote show [name]` | `git remote show` | Show remote status |
| `bit remote repair [name]` | — | Verify and repair files against remote |
| `bit push [<remote>]` | `git push [<remote>]` | Push to specified or default remote |
| `bit push -u <remote>` | `git push -u <remote>` | Push and set upstream tracking |
| `bit push --set-upstream <remote>` | `git push --set-upstream <remote>` | Push and set upstream tracking (alt) |
| `bit pull [<remote>]` | `git pull [<remote>]` | Pull from specified or default remote |
| `bit pull <remote> --accept-remote` | — | Pull from remote, accept remote state |
| `bit pull --accept-remote` | — | Accept remote file state as truth |
| `bit pull --manual-merge` | — | Interactive per-file conflict resolution |
| `bit fetch [<remote>]` | `git fetch [<remote>]` | Fetch metadata from specified or default remote |
| `bit verify` | — | Verify local files match committed metadata |
| `bit verify --remote` | — | Verify remote files match committed remote metadata |
| `bit fsck` | `git fsck` | Check integrity of internal metadata repository |
| `bit merge --continue` | `git merge --continue` | Continue after conflict resolution |
| `bit merge --abort` | `git merge --abort` | Abort current merge |
| `bit branch --unset-upstream` | `git branch --unset-upstream` | Remove tracking config |
| `bit --remote <name> init` | — | Create empty bundle on remote (ephemeral) |
| `bit --remote <name> add <path>` | — | Scan remote, write metadata, auto-commit, push bundle (ephemeral) |
| `bit --remote <name> commit -m <msg>` | — | Commit in ephemeral workspace and push bundle |
| `bit --remote <name> status` | — | Scan remote and show status including untracked files (read-only, ephemeral) |
| `bit --remote <name> log` | — | Show remote workspace history (read-only, ephemeral) |
| `bit @<remote> <cmd>` | — | Shorthand for `--remote` (needs quoting in PowerShell) |

---

## Upstream Tracking (Git-Standard Behavior)

**IMPORTANT**: bit follows git's upstream tracking conventions exactly:

1. **`bit remote add <name> <url>` does NOT set upstream** — unlike old bit behavior, adding a remote (even "origin") does not auto-configure `branch.main.remote`. This matches git.

2. **`bit push -u <remote>` sets upstream** — the `-u` / `--set-upstream` flag pushes and configures `branch.main.remote = <remote>` in one operation. This is the standard way to establish upstream tracking.

3. **Commands accept explicit remote names**:
   - `bit push <remote>` — push to named remote (no tracking change)
   - `bit pull <remote>` — pull from named remote (no tracking change)
   - `bit fetch <remote>` — fetch from named remote (no tracking change)

4. **Default remote selection**:
   - If `branch.main.remote` is set, it's used as the default
   - If not set and "origin" exists, **`bit push` uses it as fallback** (git-standard behavior)
   - If not set and "origin" exists, **`bit pull` and `bit fetch` require explicit remote** (no fallback)
   - If neither upstream nor "origin", commands fail with error suggesting `bit push <remote>` or `bit push -u <remote>`

5. **First pull does NOT set upstream**: When pulling for the first time (unborn branch), `checkoutRemoteAsMain` uses `git checkout -B main --no-track refs/remotes/origin/main`. This prevents automatic upstream tracking setup. Users must use `bit push -u <remote>` to explicitly configure tracking.

6. **Internal git remote vs upstream tracking**: bit's internal git repo has a remote named "origin" (used for fetching refs from bundles), but this is distinct from upstream tracking config (`branch.main.remote`). The internal remote is set up automatically; upstream tracking is never automatic. This distinction is critical: `Git.setupRemote` configures the internal git remote (required for bundle operations), while `Git.setupBranchTracking` sets `branch.main.remote` (must only be called from `push -u`).

This makes bit's remote behavior predictable for git users: explicit tracking setup via `-u`, explicit remote selection via argument, sensible defaults when configured, and git-standard fallback to "origin" for push operations.

---

## Remote Synchronization (Two-Phase, Action-Based)

### Key Insight: Diff-Based Sync, Not Blind Sync

We do **NOT** use `rclone sync`. Instead:

1. Compute diff between current state and desired state
2. Generate minimal action list (Copy, Move, Delete)
3. Execute actions via rclone

This saves bandwidth. For example: renaming a 1GB file becomes `rclone moveto` instead of delete + upload.

### Sync Order (CRITICAL)

**On Push:**
1. **First**: Sync files via rclone (content must exist before metadata claims it does)
2. **Then**: Push metadata bundle via rclone

**On Pull:**
1. **First**: Fetch metadata bundle via rclone
2. **Then**: Git operation (merge or checkout) updates `.bit/index/`
3. **Then**: Mirror index changes to working directory (download binaries, copy text from index)

**Rationale**: Push files first so the remote is never in a state where metadata references missing content. Pull metadata first so we know what content to fetch. After the git operation, the index is authoritative — we only need to bring actual files into alignment.

### Remote Types and Device Resolution

bit supports two kinds of remotes:

- **Cloud remotes**: rclone-based (e.g., `gdrive:Projects/foo`). Identified by URL. Uses bundle + rclone sync.
- **Filesystem remotes**: Local/network paths (USB drives, network shares, local directories). Creates a **full bit repository** at the remote location.

The `bit.Device` module handles remote type classification and device resolution:
- `RemoteType`: `RemoteFilesystem` (fixed paths), `RemoteDevice` (removable/network drives), `RemoteCloud` (rclone-based)
- `isFilesystemType`: True for both `RemoteFilesystem` and `RemoteDevice` (both use direct git operations)
- Physical storage: Identified by UUID + hardware serial (survives drive letter changes)
- Network storage: Identified by UUID only
- Each volume can have a `.bit-store` file at its root containing its UUID
- Remote configs stored in `.bit/remotes/<name>` with typed format:
  - `type: filesystem` (path lives only in git config as named remote)
  - `type: cloud\ntarget: gdrive:Projects/foo` (rclone path for transport)
  - `type: device\ntarget: black_usb:Backup` (device identity for resolution)

All remote types get a **named git remote** inside `.bit/index/.git`:
- Filesystem: `git remote add dok1 /path/to/remote/.bit/index`
- Cloud: `git remote add backup .git/fetched_remote.bundle` (bundle as URL)
- Device: `git remote add usb1 /mnt/usb/.bit/index` (URL updated at operation time)

The `bit.Remote` module provides type-aware resolution via `resolveRemote`:
```
resolveRemote :: FilePath -> String -> IO (Maybe Remote)
-- Dispatches on RemoteType:
--   RemoteFilesystem → reads URL from git config, strips .bit/index suffix
--   RemoteDevice     → resolves device UUID to mount path
--   RemoteCloud      → reads target from remote file
--   Nothing          → backward-compat fallback (infers type from old format)
```

### Transport Strategies

The transport strategy is determined by `RemoteType` classification:

```
Device.readRemoteType / isFilesystemType
  ├── RemoteCloud      → Cloud transport (bundle + rclone, existing flow)
  ├── RemoteDevice     → Filesystem transport (full repo at remote)
  └── RemoteFilesystem → Filesystem transport (full repo at remote)
```

#### Cloud Transport (Bundle + Rclone)

For cloud remotes (Google Drive, S3, etc.), bit uses **dumb storage**:
- Metadata is serialized as a Git bundle and uploaded via rclone
- Files are synced via rclone copy/move/delete operations
- The remote is just a directory of files — no Git repo, no bit commands work there

#### Filesystem Transport (Full Repo)

For filesystem remotes, bit creates a **complete bit repository** at the remote:
- The remote has `.bit/index/.git/` just like a local repo
- Anyone at the remote location can run `bit status`, `bit log`, `bit commit`, etc.
- No bundles needed — Git can talk directly repo-to-repo via `git fetch /path/to/other/.bit/index/.git/`

**Key insight**: Bundles exist to serialize git history over dumb transports that can only copy files. With filesystem access, git speaks its native protocol.

#### Filesystem Push Flow

```
filesystemPush :: FilePath -> Remote -> IO ()
```

1. **First push (no `.bit/` at remote)**: Initialize a bit repo at the remote via `initializeRepoAt`
2. **Fetch local into remote**: `git -C remote/.bit/index fetch local/.bit/index/.git main:refs/remotes/origin/main`
3. **Fast-forward check**: Verify remote HEAD is ancestor of what we're pushing (`git merge-base --is-ancestor`)
4. **Merge at remote**: `git -C remote/.bit/index merge --ff-only refs/remotes/origin/main`
5. **Sync files**: Copy changed files from local working tree to remote working tree
   - Text files: Copy from remote's updated index (git put the content there)
   - Binary files: Copy from local working tree (metadata in index, content in working tree)
6. **Update tracking ref**: Set local `refs/remotes/origin/main` to current HEAD

If the fast-forward check fails, the remote has diverged:
```
error: Remote has local commits that you don't have.
hint: Run 'bit pull' to merge remote changes first, then push again.
```

#### Filesystem Pull Flow

```
filesystemPull :: FilePath -> Remote -> PullOptions -> IO ()
```

1. **Fetch remote into local**: `git -C local/.bit/index fetch remote/.bit/index/.git main:refs/remotes/origin/main`
2. **Proof of possession**: Verify remote working tree matches remote metadata (unless `--accept-remote`)
3. **Build filesystem transport**: Create a `FileTransport` that copies files directly from remote working tree
4. **Merge locally**: Delegate to unified `pullLogic` or `pullAcceptRemoteImpl`
   - Note: Upstream tracking (`branch.main.remote`) is NOT auto-set; user must use `bit push -u <remote>`
5. **Sync files**: The unified logic uses the filesystem transport to copy files
   - Text files: Copy from local index (git merged the content there)
   - Binary files: Copy from remote working tree
6. **Update tracking ref**: Set `refs/remotes/origin/main` to the remote's HEAD hash

**Key change**: Filesystem pull now uses the same merge orchestration as cloud pull,
just with a different `FileTransport`. The merge follows the same patterns:
- First pull (unborn branch): `checkoutRemoteAsMain` (with `--no-track` to prevent auto-setting upstream) then `transportSyncAllFiles`
- Normal: `git merge --no-commit --no-ff` then `applyMergeToWorkingDir transport cwd oldHead`
- Conflicts: Same `Conflict.resolveAll` flow with (l)ocal/(r)emote choices
- `--accept-remote`: Force-checkout (with `--no-track`) then sync files via transport

This eliminates the previous duplication where `filesystemPullNormal` and
`filesystemPullAcceptRemote` reimplemented the merge flow separately.

#### Text vs Binary File Sync

For filesystem remotes, file sync distinguishes text from binary by examining the metadata file content:

```haskell
isTextMetadataFile :: FilePath -> IO Bool
-- Returns True if file exists and does NOT contain "hash: " line
-- Text files: metadata IS the content (stored directly in index)
-- Binary files: metadata contains "hash: " and "size: " (pointer to actual file)
```

- **Text files**: Content lives in `.bit/index/path`. After git merge/checkout, copy from index to working tree.
- **Binary files**: Content lives in working tree. Copy from source working tree to destination working tree.

#### The `git push` Antipattern

Do NOT use `git push` to a non-bare repo. Git refuses to update the checked-out branch:
```
error: refusing to update checked-out branch: refs/heads/main
```

The correct approach: Have the remote **fetch** from local, then **merge --ff-only**. This is what filesystem push does.

---

## Remote-Targeted Commands (`@<remote>` / `--remote <name>`)

### Problem Statement

When a user has large amounts of data already on a remote (e.g., 10GB of files on Google Drive), the traditional workflow requires:
1. Download everything locally (expensive, time-consuming)
2. Run `bit init`, `bit add`, `bit commit` locally
3. Push everything back to the remote (redundant upload)

This is wasteful when the files are already at the destination. The user wants to **create metadata for files already on the remote** without downloading them.

### Solution: Ephemeral Remote Workspaces

The `--remote <name>` flag (or its `@<remote>` shorthand) allows commands to operate against a remote as if it were a working directory, while only downloading small files (for text classification). Large binary files stay on the remote — bit just reads their hashes from `rclone lsjson --hash`.

**Key architectural property**: Each command is fully ephemeral. The workflow for every command is:
1. Fetch the bundle from the remote
2. Inflate it into a temporary directory
3. Operate on it (scan, add, commit, status, log, ls-files)
4. Re-bundle the changes (if any)
5. Push the new bundle back to the remote
6. Clean up the temporary workspace

No persistent local workspace state exists between commands. The remote's `.bit/bit.bundle` is the sole source of truth.

### The `--remote` Flag

Two equivalent syntaxes exist for specifying a remote target:

```bash
bit --remote origin init      # portable — works in all shells
bit @origin init              # shorthand — needs quoting in PowerShell
```

**Why `--remote` exists**: The `@<remote>` prefix doesn't work in PowerShell because `@` is the splatting operator. PowerShell interprets `@gdrive` as splatting the variable `$gdrive`, which is undefined, so the argument is silently dropped. `--remote <name>` is the portable alternative that works everywhere.

**Placement**: Both `--remote <name>` and `@<remote>` must appear as the first argument(s) to `bit`. This is consistent between the two forms and avoids ambiguity with subcommand flags (e.g., `bit verify --remote` uses `--remote` as a boolean flag for the `verify` command, not as a remote target).

**`--remote` is recommended** for scripts and cross-shell compatibility. `@<remote>` remains available as a convenient shorthand for interactive use in bash/zsh/cmd.

### User Workflow

```bash
# Files already exist on gdrive:Projects/footage (10GB of video + some .txt/.md)
bit init                                    # local repo (empty working dir)
bit remote add origin gdrive:Projects/footage

bit --remote origin init                    # create empty bundle on remote
bit --remote origin add .                   # scan remote, write metadata, auto-commit, push bundle

bit pull                                    # pull metadata locally (instant — just the bundle)
# Working dir is still empty, but bit knows about all 847 files
# User can then selectively download, or bit pull will sync everything
```

### Architecture

#### The `@<remote>` / `--remote <name>` Prefix

Both forms are parsed in `Commands.hs` by `extractRemoteTarget` before command dispatch. When present, it switches the execution context from "local working directory" to "remote-targeted workspace."

The remote name is resolved via the existing `resolveRemote` function, same as named remotes for push/pull.

#### Ephemeral Workspace Pattern

There is no persistent workspace on disk. Each command creates a temporary directory in the system temp folder, operates, and cleans up:

```
%TEMP%/
├── bit-remote-init/    ← used by 'init' (one-time)
├── bit-remote-ws/      ← used by 'add', 'commit' (read-write)
│   ├── bit.bundle      ← fetched bundle
│   ├── workspace/      ← inflated git repo
│   │   ├── .git/
│   │   └── <metadata>
│   └── new.bundle      ← re-bundled after changes
└── bit-remote-ro/      ← used by 'status', 'log' (read-only)
```

All temporary directories are exception-safe via `bracket` — cleanup runs even if the command fails. Leftover directories from previous runs are removed at setup to prevent collisions.

#### Bundle Inflation

Inflating a bundle into a workspace uses a three-step sequence designed to avoid Git pitfalls:

```
git init --initial-branch=main
git fetch <bundle> +refs/heads/*:refs/remotes/bundle/*
git reset --hard refs/remotes/bundle/main
```

**Why this specific sequence:**
- Fetching into `refs/remotes/bundle/*` (not `refs/heads/*`) avoids Git's "refusing to fetch into checked out branch" error
- `git reset --hard` (not `checkout -B`) ensures the working tree is populated — `checkout -B` can skip the working tree update when already on the same branch name
- `git clone` was avoided because on Windows, `removeDirectoryRecursive` can fail to fully clean up temp directories due to file locking, causing "directory already exists" errors

#### Text File Classification Without Full Download

The `rclone lsjson --hash` scan returns hashes for all files, but `fContentType = BinaryContent` for everything (rclone can't classify text without reading content). bit's text classification needs:
1. File size (available from rclone scan)
2. File extension (available from path)
3. First 8KB of content (for NULL-byte and UTF-8 checks)

Strategy:
- Files above `textSizeLimit` (default 1MB) → binary, no download needed
- Files with `binaryExtensions` (`.mp4`, `.zip`, etc.) → binary, no download needed
- Remaining small files → download to temp dir, classify with existing `hashAndClassifyFile`

For a typical 10GB media repo, this downloads maybe 50KB of text files while skipping the 10GB of video.

### Supported Commands

| Command | Behavior |
|---------|----------|
| `bit --remote <name> init` | Create empty bundle on remote (no scan — just initializes history) |
| `bit --remote <name> add <path>` | Fetch bundle → scan remote → classify files → write metadata → auto-commit → push bundle |
| `bit --remote <name> commit <args>` | Fetch bundle → commit with provided args → push bundle (useful for amending) |
| `bit --remote <name> status` | Fetch bundle → scan remote → write metadata → show git status (read-only, no push). Shows untracked files that exist on remote but aren't committed. |
| `bit --remote <name> log` | Fetch bundle → show git log (read-only, no push) |

The `@<remote>` shorthand is equivalent (e.g., `bit @origin init`).

All other commands are not supported in remote context (e.g., `bit --remote origin push` will error).

### Implementation Details

Located in `Bit.RemoteWorkspace`:

#### Core Helpers

- **`withTempDir`**: Creates a named temp directory under `%TEMP%`, exception-safe via `bracket`. Removes leftover from previous runs at setup, cleans up on exit.
- **`withRemoteWorkspace`**: Fetches bundle → inflates → runs action → if HEAD changed, re-bundles and pushes → cleans up. For read-write commands (`add`, `commit`).
- **`withRemoteWorkspaceReadOnly`**: Fetches bundle → inflates → runs action → cleans up (no push). For read-only commands (`status`, `log`).

#### 1. `bit --remote origin init` (or `bit @origin init`)

Does NOT use `withRemoteWorkspace` — there's no bundle to fetch yet.

1. Creates a temp directory
2. Checks if bundle already exists on remote (errors if it does)
3. `git init --initial-branch=main` in the temp workspace
4. `git commit --allow-empty -m "Initial remote repository"`
5. `git bundle create` from the workspace
6. Pushes bundle to remote at `.bit/bit.bundle` via rclone
7. Cleans up temp directory

#### 2. `bit --remote origin add <path>` (or `bit @origin add .`)

Uses `withRemoteWorkspace` (read-write):

1. Fetches and inflates the bundle
2. **Scans** the remote via `rclone lsjson --hash --recursive`
3. **Classifies** files into binary (by size/extension) and text candidates
4. Downloads text candidates, classifies via `hashAndClassifyFile`
5. **Clears** the workspace (removes all files except `.git`)
6. **Writes** metadata: text files get their content downloaded from remote, binary files get `hash:/size:` metadata
7. `git add` the specified paths (or `.`)
8. If changes exist, auto-commits with "Update remote metadata"
9. Re-bundles and pushes if HEAD changed

The scan+classify+write step happens on **every** `add` call — the workspace starts fresh each time, so the full remote state is reconstructed.

#### 3. `bit --remote origin commit`

Uses `withRemoteWorkspace` (read-write). Passes args through to `git commit` in the ephemeral workspace. Useful for amending the last commit message (`--amend -m "Better message"`).

#### 4. `bit --remote origin status`

Uses `withRemoteWorkspaceReadOnly`. Scans the remote via `scanAndWriteMetadata` (same as `add`) to detect untracked files, then runs `git status`. After writing metadata, runs `git add -u && git reset HEAD` to update index stat info without staging changes (prevents false "modified" reports due to stat-dirty optimization). No changes are pushed back.

#### 5. `bit --remote origin log`

Uses `withRemoteWorkspaceReadOnly`. Passes args through to `git log` in the ephemeral workspace. No changes are pushed back.

### Integration with Normal Pull

After `bit --remote origin add .`, the remote has a bundle at `.bit/bit.bundle`. A local `bit pull` will:
1. Fetch the bundle (existing `fetchBundle` logic)
2. Import it to local `.bit/index/.git/`
3. Merge the metadata
4. Sync files as needed (download from remote or copy from index)

The existing pull flow handles this transparently.

### Error Handling

- **Bundle fetch fails (no bundle)**: `withRemoteWorkspace` and `withRemoteWorkspaceReadOnly` print "fatal: no bit repository on remote. Run 'bit @remote init' first." and return `ExitFailure 1`.
- **Network error**: Propagated with "fatal: network error: ..." message.
- **Push fails**: `bundleAndPush` prints "fatal: failed to push bundle to remote." and exits.
- **Init when already initialized**: Checks for existing bundle first, errors with "fatal: remote already has a bit repository."
- **Temp directory cleanup**: `bracket` ensures cleanup runs on all code paths (success, failure, exception).

### Edge Cases

- **Remote is empty**: `scanAndWriteMetadata` finds zero files, auto-commit has nothing to stage — reports "Nothing to add."
- **Network failure during classification**: Failed downloads treated as binary, warning printed
- **Filesystem remotes**: `--remote <name>` works for filesystem remotes too, though less useful (user could just `cd` there)
- **Both `@<remote>` and `--remote` specified**: Error with clear message
- **`--remote` without argument**: Error explaining that a name is required

### Limitations

This feature does NOT provide:
- **Selective file download** — that's a separate feature (sparse working tree). After `bit pull`, all files are expected locally.
- **Incremental remote re-scan** — `bit --remote origin add` always scans from scratch (the workspace is ephemeral).
- **`bit --remote origin push`** — pushing *to* a remote workspace doesn't make sense. Push targets the actual remote.
- **Conflict resolution in remote context** — not needed. The workspace is single-writer (the local user).
- **`--remote` after the subcommand** — `--remote <name>` must appear before the subcommand (first args). This avoids ambiguity with subcommand flags like `bit verify --remote`.

---

## IO Safety and Concurrency

### Strict IO for Windows Compatibility

**Problem**: Lazy IO on Windows causes "permission denied" and "file is locked" errors due to file handles remaining open until garbage collection. When scanning hundreds of files concurrently, this manifests as random failures.

**Solution**: Eliminate all lazy IO operations and use strict `ByteString` operations exclusively.

### Implementation

#### 1. Strict IO Modules

**`Bit.ConcurrentFileIO`** — Drop-in replacements for `Prelude` file operations:
- `readFileBinaryStrict` — strict `ByteString.readFile` wrapper
- `readFileUtf8Strict` — strict UTF-8 text reading
- `readFileMaybe` / `readFileUtf8Maybe` — safe reading with `Maybe` return
- `writeFileBinaryStrict` / `writeFileUtf8Strict` — strict writing

All operations use `ByteString.readFile` / `ByteString.writeFile` which read/write the entire file and close the handle before returning.

**`Bit.Process`** — Strict process output capture:
- `readProcessStrict` — runs a process, strictly captures stdout and stderr as `ByteString`, returns `(ExitCode, ByteString, ByteString)`
- `readProcessStrictWithStderr` — runs a process with inherited stderr (for live progress), strictly captures stdout

Both functions:
- Use strict `Data.ByteString.hGetContents` (not lazy `System.IO.hGetContents`)
- Read stdout and stderr concurrently using `async` to avoid deadlocks when buffers fill
- Ensure all handles are closed and process is waited on in all code paths (using `bracket`)
- Prevent "delayed read on closed handle" errors that occur when using lazy IO with `createProcess`

**`Bit.ConcurrentIO`** — Type-safe concurrent IO newtype:
- Constructor is **not exported** to prevent `liftIO` smuggling
- Only whitelisted strict operations are exposed
- No `MonadIO` instance (intentional restriction)
- Provides `MonadUnliftIO` for `async` integration
- Includes concurrency primitives: `mapConcurrentlyBoundedC`, `QSem` operations

**`Bit.AtomicWrite`** — Atomic file writes with Windows retry logic:
- `atomicWriteFile` — temp file + rename pattern
- `DirWriteLock` — directory-level locking (MVar-based thread coordination)
- `LockRegistry` — process-wide lock registry for multiple workers
- Retry logic for Windows transient "permission denied" errors (up to 5 retries with exponential backoff)

#### 2. Module Updates

All lazy IO replaced with strict operations:
- `Bit/Scan.hs` — `.gitignore` reading uses strict ByteString
- `Bit/Device.hs` — `.bit-store`, device files, remote files use strict ByteString + atomic writes
- `Bit/Core.hs` — `readFileMaybe`, `writeFileAtomicE` (now truly atomic), metadata reading
- `Bit/Commands.hs` — `.bitignore` reading/writing uses strict ByteString + atomic writes
- `Internal/ConfigFile.hs` — Config reading uses strict ByteString instead of lazy Text IO

#### 3. HLint Enforcement

**`.hlint.yaml`** bans lazy IO functions project-wide:
- Banned: `Prelude.readFile`, `Prelude.writeFile`, `System.IO.hGetContents`, `System.IO.hGetLine`
- Banned: `Data.ByteString.Lazy.readFile`, `Data.Text.IO.readFile`
- Banned: Entire modules `Data.ByteString.Lazy`, `Data.Text.Lazy.IO`
- Suggests: `BS.readFile` over `readFile`, `atomicWriteFile` over `writeFile`
- Suggests: `Bit.Process.readProcessStrict` over `createProcess` for output capture

Process-specific rules:
- `System.IO.hGetContents` banned with message pointing to `Bit.Process.readProcessStrict`
- `System.IO.hGetLine` banned (reading in a loop is error-prone)

HLint errors appear in IDE and CI, preventing lazy IO from being reintroduced.

### Rationale

**Why strict ByteString?**
- Reads entire file into memory and closes handle immediately
- No lazy thunks keeping file handles open
- Predictable memory usage (acceptable for metadata files < 1MB)
- Cross-platform consistent behavior

**Why not lazy ByteString?**
- Lazy ByteString uses chunked IO, keeping handles open
- Garbage collection timing is unpredictable
- On Windows, this causes "file is locked" errors in concurrent scenarios

**Why atomic writes?**
- Crash safety — partial writes leave temp file, not corrupt destination
- Windows retry logic handles transient locking from antivirus/indexing
- Directory-level locking coordinates concurrent writes within process

**Why HLint rules?**
- Enforcement at development time (IDE warnings)
- Prevents lazy IO from being reintroduced during refactoring
- Documents the policy explicitly

### Concurrent File Scanning and Metadata Writing

The file scanner (`Bit/Scan.hs`) uses bounded parallelism for both scanning and writing:

**Scanning**:
- `QSem` limits concurrent file reads (default: `numCapabilities * 4`)
- Each file is fully read, hashed, and closed before moving to next
- Progress reporting uses `IORef` with `atomicModifyIORef'` for thread-safe updates
- Cache entries use strict ByteString read/write

**Metadata Writing** (`writeMetadataFiles`):
- Parallel execution with same bounded concurrency as scanning
- Skip-unchanged optimization: Before writing, checks if metadata already matches
  - Binary files: reads existing metadata, compares hash/size
  - Text files: compares mtime/size of source vs destination
- Three-phase write: (1) create directories sequentially, (2) create file parent directories, (3) write files in parallel
- Progress reporting shows files written, skipped count, and percentage
- Atomic writes for binary metadata using temp-file-rename pattern

### File Copy Progress Reporting

File copy operations during push/pull now have progress reporting (`Bit/CopyProgress.hs`):

**Implementation**:
- Chunked binary copy with byte-level progress (64KB chunks, strict `ByteString`)
- Small files (<1MB) use plain `copyFile` (no overhead), large files use chunked copy with progress
- Progress state (`SyncProgress`) tracks: total files, bytes total/copied, current file name
- Reporter thread updates at 100ms intervals via `IORef` (thread-safe, strict updates)
- TTY detection: shows in-place progress on terminal, silent on non-TTY (for log capture)

**Progress Display**:
- **Aggregate**: `Syncing files: 3/12 files, 5.1 GB / 18.3 GB (28%)`
- **Final summary**: `Synced 12 files (18.3 GB).`
- Human-readable byte formatting: `formatBytes` (B, KB, MB, GB, TB with 1 decimal place)

**Filesystem Remotes** (direct copy):
- `filesystemSyncAllFiles`, `filesystemSyncChangedFiles` (push)
- `filesystemSyncRemoteFilesToLocal`, `filesystemApplyMergeToWorkingDir` (pull)
- Progress: counts binary files only (text files are small and fast via index copy)
- Byte progress: sums file sizes from metadata before starting copy loop

**Cloud Remotes** (rclone):
- `syncRemoteFiles` (push), `syncRemoteFilesToLocal` (pull)
- Progress: file-count only (number of rclone actions completed / total)
- Simpler approach: tracks subprocess completion rather than byte-level progress

**Design Notes**:
- Complies with project's strict IO rules: no lazy `ByteString`, no lazy IO
- Windows compatible: uses `withBinaryFile` for chunked reads/writes
- Pattern matches existing scan progress in `Bit/Scan.hs`: `IORef` + reporter thread + TTY detection + `finally` cleanup

---

## Performance Optimizations

### Scan-on-Demand Architecture

**Problem**: The original design scanned the entire working directory *before* dispatching to a command, then maintained a growing `skipScan` whitelist of commands that don't need it. This was backwards:

1. **Wrong default** — new commands scan by default, silently wasting time if someone forgets to add them to `skipScan`
2. **Duplication** — the command must be listed in *both* `skipScan` and the `case` dispatch
3. **Fragile** — easy to miss commands (e.g., `remote add` was discovered to be missing from `skipScan`, causing 860 files to be scanned for a config-only operation)

**Solution**: Invert the logic — scan on demand, not by default. Commands are now classified into three tiers, with each command explicitly declaring its needs:

**Implementation** (`bit/Commands.hs`):

```haskell
-- Lightweight env (no scan) — for read-only commands
let baseEnv = do
        mRemote <- getDefaultRemote cwd
        return $ BitEnv cwd [] mRemote isForce isForceWithLease

-- Full env (scan + bitignore sync + metadata write) — for write commands
let scannedEnv = do
        syncBitignoreToIndex cwd
        localFiles <- Scan.scanWorkingDir cwd
        Scan.writeMetadataFiles cwd localFiles
        mRemote <- getDefaultRemote cwd
        return $ BitEnv cwd localFiles mRemote isForce isForceWithLease

case cmd of
    -- ── No env needed ────────────────────────────────────
    ["init"]              -> Bit.init
    ["remote", "add", ...] -> Bit.remoteAdd name url
    ["fsck"]              -> Bit.fsck cwd ...
    ["merge", "--abort"]  -> Bit.mergeAbort
    
    -- ── Lightweight env (no scan) ────────────────────────
    ("log":rest)          -> Bit.log rest >>= exitWith
    ("ls-files":rest)     -> Bit.lsFiles rest >>= exitWith
    ["remote", "show"]    -> baseEnv >>= \env -> runBitM env $ Bit.remoteShow Nothing
    ["verify"]            -> baseEnv >>= \env -> runBitM env $ Bit.verify Bit.VerifyLocal ...
    
    -- ── Full scanned env (needs working directory state) ─
    ("add":rest)          -> do _ <- scannedEnv; Bit.add rest >>= exitWith
    ("status":rest)       -> scannedEnv >>= \env -> runBitM env (Bit.status rest) >>= exitWith
    ("push":...)          -> scannedEnv >>= \env -> runBitM env Bit.push
    ("pull":...)          -> scannedEnv >>= \env -> runBitM env $ Bit.pull ...
```

**Key Changes**:

1. **No `skipScan` variable** — it no longer exists
2. **Lazy env builders** — `baseEnv` and `scannedEnv` are `IO BitEnv` actions, only executed when called
3. **Explicit tier assignment** — every command branch explicitly picks: no env, `baseEnv`, or `scannedEnv`
4. **Safe by default** — new commands default to not scanning (if you forget to call `scannedEnv`, you just get an empty `localFiles` list, which is harmless)
5. **Bitignore sync colocated with scan** — `syncBitignoreToIndex` only runs when `scannedEnv` is called, keeping related concerns together

**Command Classification**:

1. **No env needed**: Commands that operate on simple config or don't need any environment (`init`, `remote add`, `fsck`, `merge --abort`)

2. **Lightweight env (no scan)**: Read-only commands that read git history, index, or config without needing working directory state
   - `log`, `ls-files` — read git objects only
   - `remote show`, `remote repair` — read config/remote state
   - `verify`, `verify --remote` — compare against existing metadata

3. **Full scanned env**: Commands that need current working directory state
   - `status`, `restore`, `checkout` — need `localFiles` from env
   - `add`, `commit`, `diff` — need scan for side effects (metadata write), even though they don't use `localFiles` directly
   - `push`, `pull`, `fetch` — need working directory state for sync
   - `merge --continue` — needs working directory state to resolve conflicts

**Performance Impact**: In large repositories, read-only commands now have instant response times. The old design would scan 860 files for `bit remote add`, the new design scans zero.

---

## Verification and Consistency

### `bit verify`

Verifies local working tree files match their committed metadata. Scans the working directory to update `.bit/index/` metadata, then runs `git diff` on the index repo to find files whose metadata changed from the committed state. Any difference means the file has been corrupted or modified since the last commit.

**Why scan + git diff (not direct file-vs-metadata comparison):** An earlier approach loaded metadata from `.bit/index/` on disk and compared file hashes against it. The problem: any command that scans the working directory (`bit status`, `bit add`, etc.) updates `.bit/index/` metadata to reflect current file state. If a file was corrupted and the user happened to run `bit status` first, the metadata would be updated to match the corrupted content — and verification would pass, silently missing the corruption. By comparing against the *committed* state in git (what was explicitly committed with `bit commit`), verification is immune to stale metadata. The committed state doesn't change just because a scan ran.

**Implementation:**
1. `Scan.scanWorkingDir` + `Scan.writeMetadataFiles` — update `.bit/index/` to match current working tree
2. `git diff --name-only` in `.bit/index` repo — find files whose metadata changed from HEAD
3. For each changed binary file: read committed metadata (`git show HEAD:<path>`) for expected hash/size, read filesystem metadata for actual hash/size → `HashMismatch`
4. For each changed text file: report `HashMismatch` (git diff already proves the content changed)
5. For missing files: `git ls-tree -r HEAD` lists committed paths, check existence in working tree → `Missing`

**Progress reporting**: On TTY with >5 files, displays live progress: `Checking files: N/Total (X%)...`

### `bit verify --remote`

Detects remote type via `getRemoteType`/`Device.isFilesystemType` and routes accordingly:

- **Filesystem remotes**: Scans the remote working directory using `Verify.verifyLocalAt`, the same scan + git diff approach as local verification.
- **Cloud remotes**: Fetches the remote bundle (committed metadata), scans remote files via `rclone lsjson --hash`, and compares.

**Progress reporting**: On TTY with >5 files, displays live progress during comparison phase (cloud remotes only).

### `bit fsck`

Runs `git fsck` on the internal metadata repository (`.bit/index`). Checks the integrity of the object store — that all commits, trees, and blobs are valid and consistent. This is a passthrough to git's own integrity check. Use `bit verify` to check file integrity instead.

### `bit remote repair`

Verifies both local and remote files against their respective metadata, then repairs any broken/missing files by copying verified files from the other side using content-addressable lookup.

**Algorithm**:
1. Resolve remote, load binary metadata from both sides
2. Verify both sides: `verifyLocal` and `verifyRemote` (or `verifyLocalAt` for filesystem remotes)
3. Build content indexes from verified files (metadata entries not in the issue set)
4. For each local issue: look up expected (hash, size) in remote verified index, copy from remote
5. For each remote issue: look up expected (hash, size) in local verified index, copy to remote
6. Report summary: repaired, failed, unrepairable

**Content-addressable repair**: Files are matched by (hash, size), not by path. If `photos/song.mp3` is corrupted locally but `backup/song_copy.mp3` on the remote has the same hash and size, it will be used as the repair source.

**Output**: Full comparison report saved to `.bit/last-check.txt` for detailed analysis.

---

## Handling Remote Divergence

When remote files don't match remote metadata (detected via `bit verify --remote` or during `bit pull`):

### Resolution Option 1: Accept Remote Reality (`--accept-remote`)

Force-checkout the remote branch so git puts the correct metadata in
`.bit/index/`, then mirror the changes to the working directory. This is
architecturally identical to a normal pull — just a force-checkout instead of
a merge. Git manages the index; we only sync actual files.

The flow:
1. Fetch remote bundle (git gets remote history)
2. Record current HEAD (for diff-based sync)
3. `git checkout -f -B main --no-track refs/remotes/origin/main` (force-checkout remote without setting upstream)
4. `applyMergeToWorkingDir` (diff old HEAD vs new HEAD, sync files)
5. Update tracking ref

**Important**: `--accept-remote` must NOT scan remote files via rclone and write
metadata directly. Rclone cannot distinguish text from binary files
(`fContentType = BinaryContent` for everything), so text files would get `hash:/size:`
metadata instead of their actual content.

### Resolution Option 2: Force Local (`bit push --force`)

Upload all local files, overwriting remote. Push metadata bundle. Requires confirmation.

### Resolution Option 3: Manual Merge (`--manual-merge`)

Interactive per-file conflict resolution:
- For each conflict, displays local hash/size vs remote hash/size
- User chooses (l)ocal or (r)emote for each file
- Resolution is applied via git checkout mechanics
- Supports `bit merge --continue` and `bit merge --abort`

---

## Design Decisions

### What We Chose

1. **Phantom-typed hashes**: `Hash (a :: HashAlgo)` — the compiler distinguishes MD5 from SHA256. Mixing algorithms is a compile error. The `DataKinds` extension is used per-module.

2. **Unified metadata parser and loader**: A single `bit/Internal/Metadata.hs` module handles all parsing and serialization, and `Bit/Verify.hs` provides unified metadata loading via the `MetadataSource` abstraction. This eliminates the class of bugs where multiple parsers or loaders handle edge cases differently. The `MetadataEntry` type forces callers to explicitly handle the binary vs. text file distinction, ensuring text files (whose git blobs may have normalized line endings) are not incorrectly hash-verified across sources.

3. **`ReaderT BitEnv IO` (no free monad)**: The application monad is `ReaderT BitEnv IO`. A free monad effect system was considered (for testability and dry-run mode) but rejected as premature — no pure tests or dry-run usage existed to justify the complexity. Direct IO with `ReaderT` for environment threading is cleaner for now.

4. **Pure sync pipeline**: `diff` and `plan` are pure functions composed in `bit.Pipeline`. The intermediate `[GitDiff]` is preserved (not merged with `plan`) for display and testing. Property tests in `test/PipelineSpec.hs` verify the pipeline.

5. **Structured conflict resolution**: Conflict handling is a fold over a conflict list (`bit.Conflict`), not an imperative block. Decision logic is separated from IO mechanics.

6. **Remote as opaque type**: `Remote` is exported without its constructor. Only `remoteName` is public for display. `remoteUrl` exists for Transport to extract the URL, but business logic in Bit.hs should use `displayRemote` for user-facing messages.

7. **Tracking ref invariant**: `refs/remotes/origin/main` must always reflect
   what the remote actually has — never a local-only commit. After **push**,
   updating to HEAD is correct (the remote now has our history). After
   **pull/merge**, update to the hash from the fetched bundle, because HEAD
   includes merge commits the remote doesn't know about. Violating this
   causes the next `fetchFromBundle` to encounter a non-fast-forward update
   and silently fail to update the tracking ref, making subsequent merges
   operate against stale history.

8. **Git is the sole authority over `.bit/index/`**: No code path should write
   metadata files to the index and commit them directly. The index is always
   populated by git operations (merge, checkout, commit via `bit add`). After
   any git operation that changes HEAD, we only need to mirror those changes
   onto the actual working directory. This invariant applies to all pull paths
   including `--accept-remote`.

9. **Always commit when MERGE_HEAD exists**: After conflict resolution,
   `git commit` must always be called — never guarded by `hasStagedChanges`.
   When the user chooses "keep local" (`--ours`), the index becomes identical
   to HEAD. A `hasStagedChanges` check would skip the commit, leaving
   `MERGE_HEAD` dangling. Git's `commit` succeeds when `MERGE_HEAD` exists
   regardless of index state. Skipping the commit breaks the next push
   (ancestry check fails because HEAD was never advanced).

10. **The `oldHead` capture pattern**: Before any git operation that changes
    HEAD (merge, checkout), capture HEAD so `applyMergeToWorkingDir` can diff
    old vs new and sync only what changed. This pattern appears in `pullLogic`,
    `mergeContinue`, and `pullAcceptRemoteImpl`. The only exception is first
    pull (`oldHead = Nothing`), which falls back to `syncRemoteFilesToLocal`.

11. **Proof of possession on push/pull**: bit-lite must verify that the sender's
    working tree matches its metadata before transferring that metadata. On push,
    the local repo is verified; on pull, the remote is verified. This is necessary
    because bit-lite has no object store — the working tree is the only copy of
    binary content, and metadata without matching content is meaningless. Git and
    bit-solid don't need this rule because their object stores back up every claim
    unconditionally. The existing divergence resolution mechanisms
    (`--accept-remote`, `--force`, `--manual-merge`) serve as escape hatches when
    verification fails.

12. **Transport strategy split**: Push and pull dispatch based on `RemoteType`
    classification. Cloud remotes use bundle + rclone (dumb storage). Filesystem
    remotes use direct git fetch/merge (smart storage — full bit repo at remote).
    This split is keyed off `Device.readRemoteType`/`isFilesystemType` and happens
    in `Bit.Core.push` and `Bit.Core.pull`. **Merge orchestration is unified**: Both
    cloud and filesystem pull paths use the same `pullLogic` and
    `pullAcceptRemoteImpl` functions, parameterized by a `FileTransport` that
    abstracts how files are copied. The transport is the only difference — all
    `oldHead` capture, conflict resolution, tracking ref updates, and MERGE_HEAD
    handling is shared. This eliminates the duplication that previously existed
    where filesystem pull reimplemented the merge flow separately.

13. **Filesystem remotes are full repos**: When pushing to a filesystem path,
    bit creates a complete bit repository at the remote (via `initializeRepoAt`).
    Anyone at that location can run bit commands directly. This is the natural
    model for USB drives, network shares, and local collaboration directories.
    Bundles are skipped entirely — git talks repo-to-repo via filesystem paths.

14. **Upstream tracking requires explicit `-u` flag**: Following git-standard
    behavior, `branch.main.remote` is never set automatically. Users must use
    `bit push -u <remote>` to establish upstream tracking. This includes:
    - `bit remote add` does NOT set upstream (unlike old bit behavior)
    - First pull uses `git checkout -B main --no-track` to prevent auto-tracking
    - `bit push` falls back to "origin" if it exists and no upstream is configured
    - `bit pull` and `bit fetch` require explicit remote if no upstream is set
    This makes bit's remote behavior predictable for git users.

15. **Strict ByteString IO exclusively**: All file operations use strict
    `ByteString` (`Data.ByteString.readFile` / `writeFile`), never lazy IO
    (`Prelude.readFile`, `Data.ByteString.Lazy`, `Data.Text.Lazy.IO`). Lazy IO
    on Windows keeps file handles open until garbage collection, causing
    "permission denied" and "file is locked" errors in concurrent scenarios.
    Strict IO reads/writes the entire file and closes the handle immediately.
    HLint rules enforce this project-wide.

16. **Atomic writes with retry logic**: All important writes use the temp-file +
    rename pattern (`Bit.AtomicWrite.atomicWriteFile`). On Windows, includes
    retry logic (5 attempts with exponential backoff) to handle transient
    "permission denied" errors from antivirus and file indexing. Directory-level
    locking coordinates concurrent writers within the process.

17. **ConcurrentIO newtype without MonadIO**: For modules that need type-level
    IO restrictions, `Bit.ConcurrentIO` provides a newtype wrapper where the
    constructor is not exported. This prevents smuggling arbitrary lazy IO via
    `liftIO`. Only whitelisted strict operations are exposed. Used where the
    type system should enforce IO discipline (currently available but not widely
    adopted; `Bit.ConcurrentFileIO` with plain `MonadIO` is used in most places
    for simplicity).

18. **Ephemeral remote workspaces (no persistent state)**: Remote-targeted
    commands (`bit @remote init/add/commit/status/log/ls-files`) use an ephemeral
    workspace pattern — each command fetches the bundle from the remote,
    inflates into a system temp directory, operates, re-bundles if changed,
    pushes back, and cleans up. No persistent workspace is stored under
    `.bit/`. This design was chosen over persistent workspaces because:
    - It avoids stale state (the remote is always the source of truth)
    - It avoids disk space accumulation from workspace copies
    - It avoids Windows file locking issues with long-lived directories
    - It makes each command self-contained and idempotent
    - Cleanup is exception-safe via `bracket`
    Bundle inflation uses `git init` + `git fetch` into tracking refs +
    `git reset --hard` (not `checkout -B`, which can skip working tree
    updates when already on the target branch; not `git clone`, which
    fails on Windows when temp directories aren't fully cleaned up).

### What We Deliberately Do NOT Do

- **`RemoteState` does not need a typed state machine.** The pattern match in push logic is clear and total.
- **`FileIndex` does not need a Representable functor.** The dual indices (`byPath`/`byHash`) are an implementation detail.
- **`GitDiff` does not need a Group structure.** bit computes diffs fresh each time; inverse/compose would be dead code.
- **No Arrow syntax.** Plain `>>=` and function composition are clearer.
- **No MTL-style type classes** (`MonadGit`, `MonadRclone`). Everything is concrete.
- **No post-sync metadata rescanning.** After syncing files to the working
  directory, we do NOT re-scan and rewrite metadata. Git already put the
  correct metadata in the index. Rescanning would be redundant at best,
  wrong at worst (e.g., overwriting text file content with hash/size if the
  scan's text/binary classification differs from git's).

---

## Known Deviations and TODOs

### Remaining Work

- **Transaction logging**: For resumable push/pull operations.
- **Error messages**: Some need polish to match Git's style and include actionable hints.
- **`isTextFileInIndex` fragility**: The current check (looking for `"hash: "` prefix) works but is indirect. A more robust approach might check whether the file parses as metadata vs. has arbitrary content. Low priority since current approach works correctly.
- **Verification caching**: Cache verification results keyed on (path, mtime, size) to skip re-hashing unchanged files on subsequent push/pull operations. Would significantly speed up verification for large repos where most files haven't changed.

### Future (bit-solid)

- Content-Addressed Storage (CAS)
- Sparse checkout via symlinks to CAS blobs
- `bit materialize` / `bit checkout --sparse`

---

## Current Implementation State

### Implemented and Working

- `bit init` — creates `.bit/`, initializes Git in `.bit/index/.git`
- `bit add` — scans files, computes MD5 hashes, writes metadata, stages in Git
- `bit commit`, `diff`, `status`, `log`, `restore`, `checkout`, `reset`, `rm`, `mv`, `branch`, `merge` — delegate to Git
- `bit remote add/show/check` — named remotes with device-aware resolution
- `bit push` — Cloud: diff-based file sync via rclone, then push metadata bundle. Filesystem: fetch+merge at remote, then sync files
- `bit pull` — Cloud: fetch metadata bundle, then diff-based file sync via `applyMergeToWorkingDir`. Filesystem: fetch from remote, merge locally, sync files
- `bit pull --accept-remote` — force-checkout remote branch, then mirror changes to working directory
- `bit pull --manual-merge` — interactive per-file conflict resolution
- `bit merge --continue / --abort` — merge lifecycle management
- `bit fetch` — fetch metadata bundle only
- `bit verify` — local file verification against committed metadata (scan + git diff)
- `bit verify --remote` — remote file verification against committed remote metadata
- `bit fsck` — passthrough to `git fsck` on internal metadata repository
- `bit --remote <name>` / `bit @<remote>` — ephemeral remote workspace commands (`init`, `add`, `commit`, `status`, `log`, `ls-files`); each command fetches bundle, inflates into temp dir, operates, re-bundles if changed, pushes, and cleans up
- Pipeline: pure diff → plan → action generation with property tests
- Device-identity system for filesystem remotes (UUID + hardware serial)
- Filesystem remote transport (full bit repo at remote, direct git fetch/merge)
- Conflict resolution module with structured fold (always commits when MERGE_HEAD exists)
- Unified metadata parsing/serialization
- `oldHead` pattern for diff-based working-tree sync across all pull/merge paths
- Strict ByteString IO throughout — no lazy IO, eliminates Windows file locking issues
- Atomic file writes with Windows retry logic — crash-safe, handles antivirus/indexing conflicts
- Concurrent file scanning with bounded parallelism and progress reporting
- HLint enforcement of IO safety rules
- Proof of possession verification for push and pull operations
  - `verifyLocalAt` function for verifying arbitrary repo paths (used for filesystem remotes)
  - Integration with existing escape hatches (`--accept-remote`, `--manual-merge`)

### Module Map

| Module | Role |
|--------|------|
| `bit/Commands.hs` | CLI dispatch, env setup |
| `Bit.hs` | All business logic |
| `Internal/Git.hs` | Git command wrapper (`runGitAt`/`runGitRawAt` for arbitrary paths) |
| `Internal/Transport.hs` | Rclone command wrapper |
| `Internal/Config.hs` | Path constants |
| `Internal/ConfigFile.hs` | Config file parsing (strict ByteString) |
| `bit/Types.hs` | Core types: Hash, FileEntry, BitEnv, BitM |
| `bit/Internal/Metadata.hs` | Canonical metadata parser/serializer |
| `bit/Scan.hs` | Working directory scanning, hash computation (`hashAndClassifyFile` returns `HashWithContentType`), cache entries use `ContentType`, parallel metadata writing with skip-unchanged optimization (concurrent, strict IO). `readMetadataFile`, `getFileHashAndSize` return `Maybe MetaContent`. |
| `bit/Concurrency.hs` | Bounded parallelism helpers: concurrency level calculation, sequential/parallel mode switching |
| `bit/Diff.hs` | Pure diff: FileIndex → FileIndex → [GitDiff] |
| `bit/Plan.hs` | Pure plan: GitDiff → RcloneAction |
| `bit/Pipeline.hs` | Composed pipeline: diffAndPlan, pushSyncFiles, pullSyncFiles |
| `bit/Verify.hs` | Local and remote verification; scan + git diff approach for local (compares against committed metadata); unified metadata loading via `MetadataSource` abstraction (`FromFilesystem`, `FromCommit`); `MetadataEntry` type distinguishes binary (hash-verifiable) from text files (existence-only); `verifyLocalAt` for filesystem remotes |
| `bit/Fsck.hs` | Passthrough to `git fsck` on `.bit/index` metadata repository |
| `bit/Remote.hs` | Remote type, resolution, RemoteState, FetchResult |
| `bit/Remote/Scan.hs` | Remote file scanning via rclone |
| `bit/RemoteWorkspace.hs` | Ephemeral remote workspace: `initRemote`, `addRemote`, `commitRemote`, `statusRemote`, `logRemote`; `withRemoteWorkspace` / `withRemoteWorkspaceReadOnly` orchestration; bundle inflation via `init+fetch+reset --hard` |
| `bit/Device.hs` | Device identity, volume detection, .bit-store (strict IO, atomic writes), `RemoteType` classification, `isFixedDrive` |
| `bit/DevicePrompt.hs` | Interactive device setup prompts |
| `bit/Conflict.hs` | Conflict resolution: Resolution, DeletedSide, ConflictInfo, resolveAll |
| `bit/Utils.hs` | Path utilities, filtering, atomic write re-exports |
| `bit/AtomicWrite.hs` | Atomic file writes, directory locking, lock registry |
| `bit/ConcurrentIO.hs` | Type-safe concurrent IO newtype (no MonadIO) |
| `bit/ConcurrentFileIO.hs` | Strict ByteString file operations |
| `bit/Process.hs` | Strict process output capture (concurrent stdout/stderr reading) |
| `bit/Progress.hs` | Centralized progress reporting for terminal operations |
| `bit/CopyProgress.hs` | Progress tracking for file copy operations (push/pull sync) |

---

## Test Infrastructure

### Lint Test Suite (Pattern Safety + Format Validation)

**Purpose**: Prevent dangerous Windows environment variable patterns and shelltest format errors in test files.

**Two Categories of Violations**:

1. **Pattern Safety**: Dangerous Windows environment variables that can cause commands to escape test sandboxes
2. **Format Validation**: Shelltest Format 3 syntax violations that cause parse errors and prevent tests from running

**Problem 1 (Pattern Safety)**: Windows expands environment variables like `%CD%` before command chains execute. Example:
```batch
cd test\cli\output\work_mytest & bit remote add origin "%CD%\test\cli\output\remote_mirror"
```
If the `cd` fails, `%CD%` still expands to the current directory (potentially the main repo), causing `bit remote add` to modify the development repo's remote URL instead of the test repo's.

**Problem 2 (Format Validation)**: Shelltest Format 3 allows only one of each directive (`<<<`, `>>>`, `>>>2`, `>>>=`) per test case. Multiple directives cause parse errors that silently prevent tests from running. Example:
```batch
# WRONG - Two >>>2 directives:
command
>>>2 /error 1/
>>>2 /error 2/
>>>= 1
```
This causes a parse error and the test never executes, giving false confidence that the test is passing.

**Solution**: Automated guards enforcing both relative path usage and shelltest format rules.

#### Lint Test (Primary Guard)

**Location**: `test/LintTestFiles.hs`

**Test Suite**: `cabal test lint-tests`

**What it does**:
- Recursively scans all `.test` files under `test/cli/`
- **Pattern Safety Checks** (case-insensitive):
  - `%CD%` — current directory, expands before command execution
  - `%~dp0` — batch script directory variable
  - `%USERPROFILE%`, `%APPDATA%`, `%HOMEDRIVE%`, `%HOMEPATH%` — user directory variables
- **Format Validation Checks**:
  - Parses test cases (separated by blank lines)
  - Detects duplicate directives within a single test case:
    - Multiple `<<<` (stdin)
    - Multiple `>>>` (stdout)
    - Multiple `>>>2` (stderr)
    - Multiple `>>>=` (exit code)
  - Distinguishes between new directives and multi-line continuation
- Fails with detailed error showing file, line number, violation type, and fix
- Runs as part of `cabal test` and CI

**Example Output on Violation**:
```
DANGEROUS PATTERN in test/cli/remote-check.test:15
  Found: %CD%
  Line:  cd test\cli\output\work_mytest & bit remote add origin "%CD%\test\cli\output\remote_mirror"

  Why dangerous: Windows expands %CD% before the command chain executes.
  If the preceding `cd` fails, commands run in the main repo directory.
  Fix: Use relative paths (e.g., ..\remote_mirror) instead.
```

#### Pre-commit Hook (Secondary Guard)

**Location**: `scripts/pre-commit`

**Installation**: `scripts\install-hooks.bat` (run once after cloning)

**What it does**:
- Scans only staged `.test` files before commit
- Checks for the same dangerous patterns
- Blocks commit if violations found
- Provides clear error message with fix guidance

**Note**: Optional developer convenience; the lint test is the primary enforcement mechanism.

#### Forbidden Patterns List

The patterns are centrally defined in `test/LintTestFiles.hs`:
- `%CD%` — expands to current directory before command chains execute
- `%~dp0` — batch script directory, same timing issue
- `%USERPROFILE%` — could resolve to real user directories
- `%APPDATA%` — could resolve to real app data directories
- `%HOMEDRIVE%` / `%HOMEPATH%` — could resolve to real user paths

**Correct Pattern**: Use relative paths that resolve at command execution time:
```batch
# WRONG (banned):
cd test\cli\output\work_mytest & bit remote add origin "%CD%\test\cli\output\remote_mirror"

# CORRECT:
cd test\cli\output\work_mytest & bit remote add origin ..\remote_mirror
```

#### Documentation

See `test/cli/README.md` "Forbidden Patterns" section for detailed explanation and examples.

---

## Guardrails

**DO NOT:**
- Reintroduce a Manifest abstraction (we removed it intentionally)
- Store content in Git (only metadata or text files in the index)
- Use `rclone sync` — use action-based sync with explicit operations
- Add fields to metadata beyond `hash` and `size`
- Track symlinks or empty directories
- Implement CAS yet (that's bit-solid, mark as TODO)
- Add MTL-style type classes or free monad effects (premature)
- Merge `diff` and `plan` into a single function
- Write metadata to `.bit/index/` directly and then commit (bypasses git;
  rclone scans set `fContentType = BinaryContent` for everything, producing wrong metadata
  for text files)
- Guard merge commits on `hasStagedChanges` when `MERGE_HEAD` exists (the
  commit must always be created to finalize the merge, even when the tree is
  unchanged — e.g., "keep local" resolution)
- Re-scan the working directory after sync to "fix" metadata (the index is
  already correct after the git operation; rescanning is redundant or harmful)
- Use `git push` to a filesystem remote (git refuses to update checked-out
  branches; use fetch+merge at the remote instead)
- Auto-set upstream tracking (`branch.main.remote`) on pull, fetch, or remote
  add operations — this must only be done via explicit `bit push -u <remote>`
- Use lazy IO (`Prelude.readFile`, `writeFile`, `hGetContents`, `Data.ByteString.Lazy`,
  `Data.Text.Lazy.IO`) — causes "file is locked" errors on Windows; use strict
  ByteString operations exclusively
- Use `createProcess` with `System.IO.hGetContents` for capturing process output —
  causes "delayed read on closed handle" errors; use `Bit.Process.readProcessStrict` instead
- Use plain `writeFile` for important files — use `atomicWriteFile` for crash safety
  and Windows compatibility
- Push metadata from an unverified working tree (the metadata may reference
  files that are missing or corrupted — see Proof of Possession rule)
- Pull metadata from an unverified remote (corruption propagates through
  metadata; verify remote first, suggest `--accept-remote` if verification fails)
- Store persistent remote workspace state under `.bit/` — remote-targeted
  commands must use ephemeral temp directories (fetch → inflate → operate →
  push → cleanup). The remote bundle is the sole source of truth.
- Use `git checkout -B` in bundle inflation — it can skip the working tree
  update when already on the target branch; use `git reset --hard` instead
- Use `git clone` in bundle inflation on Windows — temp directory cleanup may
  not fully complete due to file locking, causing "directory already exists"
  errors; use `git init` + `git fetch` + `git reset --hard` instead
- Create `.bit/` from non-init code paths — only `init` and `initializeRepoAt`
  may create `.bit/` from scratch. All other functions (`scanWorkingDir`,
  `writeMetadataFiles`, `saveCacheEntry`) must check that `.bit/` already
  exists and silently no-op if it doesn't. This prevents accidental `.bit/`
  directory creation in non-repo directories (e.g., from Windows `&`-chaining
  after a failed `cd`)

**ALWAYS:**
- Prefer `rclone moveto` over delete+upload when hash matches
- Push files before metadata, pull metadata before files (cloud remotes)
- For filesystem remotes, use fetch+merge (not `git push` to non-bare repos)
- Use `Bit.AtomicWrite.atomicWriteFile` for all important file writes (temp file + rename pattern)
- Use strict `ByteString` operations for all file IO — never `Prelude.readFile`, `writeFile`, or lazy ByteString/Text
- Use `Bit.ConcurrentFileIO.readFileBinaryStrict` / `readFileUtf8Strict` for reading
- Match Git's CLI conventions and output format
- Keep Transport dumb — no domain knowledge in Transport
- Keep Git.hs dumb — no domain interpretation
- All business logic in Bit.hs
- Use the unified metadata parser from `bit/Internal/Metadata.hs`
- After pull/merge, set refs/remotes/origin/main to the bundle hash, not HEAD
- Capture `oldHead` before any git operation that changes HEAD, then use
  `applyMergeToWorkingDir` to sync the working directory
- Let git manage `.bit/index/` — all pull paths (normal, `--accept-remote`,
  `--manual-merge`, `mergeContinue`) must update the index via git operations
  (merge, checkout), never by writing files directly
- Always call `git commit` after conflict resolution when `MERGE_HEAD` exists
- Update tracking ref after filesystem pull (same invariant as cloud pull)
- Use `--no-track` flag for any `git checkout` that should not set upstream
  tracking (e.g., `checkoutRemoteAsMain`, `--accept-remote` flows)
- Verify local working tree before push (proof of possession)
- Verify remote before pull (proof of possession); for cloud remotes use
  `rclone lsjson --hash` (free on Google Drive), for filesystem remotes hash
  the files
- When verification fails, refuse the operation and suggest resolution:
  `bit add` / `bit restore` for local issues, `--accept-remote` / `--force` /
  `--manual-merge` for remote issues
```

---

## docs/verify-and-repair-tutorial.md

**Path:** `docs/verify-and-repair-tutorial.md`

*Source file.*

```markdown
# Verify and Repair Tutorial

## Why Verification Matters

In Git, every blob is stored by its SHA-1 hash. If you have the metadata, you have
the content — they're the same thing.

bit is different. Binary files live in the working tree as regular files. The
metadata *claims* that `photos/vacation.jpg` has a certain hash and size — but the
file could have been silently corrupted by a bad disk sector, accidentally
overwritten, or partially transferred. Verification is how bit checks that reality
matches the claims.

## `bit verify`

Checks whether the files in your working tree match what you committed.

```
$ bit verify
[OK] All 47 files match metadata.
```

Every tracked file is hashed and compared against the committed metadata. If
anything doesn't match, bit tells you exactly what's wrong:

```
$ bit verify
[ERROR] Hash mismatch: data/model.bin
  Expected: md5:a1b2c3d4e5f6...
  Actual:   md5:9f8e7d6c5b4a...
Checked 47 files. 1 issues found. Run 'bit status' for details.
```

bit also checks for missing files — tracked files that no longer exist on disk.

### Automatic Verification

bit verifies automatically in:

- **`bit push`** verifies local files before pushing — you can't push metadata
  that doesn't match your actual files
- **`bit pull`** verifies remote files before pulling — you won't pull corrupted
  data from a remote

This is the **proof of possession** rule: you can't transfer metadata claims you
can't substantiate, i.e. provide the actual file.

## `bit verify --remote`

Same idea, but checks the files on the remote:

```
$ bit verify --remote
[OK] All 47 files match metadata.
```

For cloud remotes (Google Drive, S3, etc.), this is fast — cloud providers store
MD5 hashes as native file metadata, so bit can verify without downloading anything.

For filesystem remotes (USB drives, network shares), bit reads and hashes every
file on the remote, same as local verification.

## `bit remote repair`

When verification finds problems, repair fixes them. It checks both sides, then
copies healthy files to replace broken ones.

```
$ bit remote repair
Repairing against remote: gdrive (gdrive:MyBackup)

Verifying local files...
  47 files checked, 1 issues
Verifying remote files...
  47 files checked, 0 issues

Repairing 1 file(s)...
  [REPAIRED] data/model.bin

1 repaired, 0 failed, 0 unrepairable.
```

### How Repair Finds the Right File

Repair matches files by **content hash and size**, not by path. This means:

- If `photos/vacation.jpg` is corrupted locally, but the remote has an identical
  copy at `backup/vacation_copy.jpg`, it will be used as the repair source.
- Renamed files, moved files, and duplicates all serve as valid repair sources.

Two files with the same hash and size have identical content — it doesn't matter
what they're called or where they live.

### Repair Works Both Ways

Repair isn't one-directional. If a local file is broken, it copies from the remote.
If a remote file is broken, it copies from local. It fixes whatever it can on
both sides in a single run.

### When Files Can't Be Repaired

A file is **unrepairable** when the same content is broken on both sides:

```
$ bit remote repair
  [UNREPAIRABLE] data/model.bin

0 repaired, 0 failed, 2 unrepairable.
```

If this happens, you'll need to restore the file from another source (another
backup, the original file, etc.), then `bit add` and `bit commit` to update
the metadata.

## `bit fsck`

Runs `git fsck` on bit's internal metadata repository. This checks that the
metadata store itself is healthy — not the files, but the database that tracks
them. Rarely needed; it's the "something is really wrong with my disk" command.

```
$ bit fsck
```

## Putting It All Together

A typical workflow when something seems wrong:

```bash
# 1. Check if local files are healthy
bit verify

# 2. Check if remote files are healthy
bit verify --remote

# 3. If either side has issues, repair automatically
bit remote repair

# 4. Verify again to confirm everything is clean
bit verify
```
```

---

