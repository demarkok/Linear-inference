# Generic bidirectional syntax

### Terms and metaterms

First, we have a block defining metavariables (lhs) and term constructors (rhs).
```
        x ::= a() | b() | c()
        X ::= A() | B() | C()
        t ::= var(x⁺) | λ(x⁺, t⁺) | @(t⁺ , t⁺) | :(t⁺, T⁺)
        T ::= Var(X⁺) | ->(T⁺, T⁺)
        Γ ::= ·() | cons⇒(x⁺, T⁺, Γ⁺) | cons⇐(x⁻, T⁻, Γ⁺)
```

In general, a definition of a metavariable `M` has the following form

```
M ::= T_1(M_{1,1}±, M_{1,2}±, …, M_{1,k_1}±) | … | T_n(M_{m,1}±, M_{m,2}±, …, M_{m,k_m}±)
```

Here `T_i` are term constructors, each of which is used exactly once. `M_{i,j}` are metavariables (that must be defined in the block).
Every argument of the term constructors is marked as either `+` (input) or `-` (output). 


It gives us a set of *`M`-terms* and a set of *`M`-metaterms* (under-constructed terms) for every metavariable `M`.

For instance:

```
x-terms: {a(), b(), c()}
t-terms: {var(a()), λ(a(), var(b())), @(var(a()), var(a())), ...}
t-metaterms: t-terms ∪ {t, var(x), λ(a(), @(t, t)), ...}
```

Terms correspond to an initial algebra of a polynomial functor.
What do metaterms correspond to categorically. Containers? Presheaves? Free Monads?


## Positive

Now let us assume the term constructors only have *positive arguments*.

In particular, we do not allow `cons⇐(x⁻, T⁻, Γ⁺)`.
It means  we don't embed "outputs" into the terms, and deal with them at the level of judgments.

Each M-metaterm can be interpreted in two ways:
- as a producer/builder. Assume we have initialized `X` and `T` with some terms.
    then metaterm `->(T, ->(Var X, Var X))` builds a `T`-term by substitution;
- as a consumer/pattern: a `T`-term can be matched against `->(T, ->(Var X, Var X))`,
        which, if successful, initializes `X` and `T`. Notice that in this case it
        also *unifies* the "arguments" of `->(Var X, Var X)`.

### Judgments

Now we define a set of judgments and inference rules.
Each judgment declaration has form 
`J(M1±, M2±, …, Mn±)` where `Mi` is a metavariable marked as either `+` or `-`. If it is marked as `-`, it is considered as an input, otherwise as an output.
For each judgment, there is a number of inference rules.
An inference rule for a judgment `J(M1±, M2±, …, Mn±)` has form

```
J1(N1-mt, N2-mt, …, Nk-mt), …, Jl(Q1-mt, Q2-mt, …, Qn-mt)
-----------------------------------------------------------
J(M1-mt, M2-mt, …, Mn-mt)
```

The precondition of the rule is a conjunction of judgments applied to metaterms.
We require these application to be consistent with the declarations. 
For example, we require `J1` to be declared as `J1(N1±, …, Nk±)`.

### Example

It would be standard rules for lambda calculus, if we used infix notation.


```
    ⊢⇒(Γ⁺, t⁺, T⁻)


    ------------------------------
    ⊢⇒(cons⇒(x, T, Γ), var(x), T)


    ⊢⇒(Γ, var(x), T)
    -------------------------------
    ⊢⇒(cons⇒(y, T', Γ), var(x), T)


    ⊢⇒(Γ, t1, →(A, B))   ⊢⇐(Γ, t2, A)
    -----------------------------------
    ⊢⇒(Γ, @(t1, t2), B)


    ⊢⇐(Γ, t, T)
    ------------------
    ⊢⇒(Γ, :(t, T), T)
```

```
    ⊢⇐(Γ⁺, t⁺, T⁺)


    ⊢⇒(Γ, t, T)
    ------------
    ⊢⇐(Γ, t, T)


    ⊢⇐(cons⇒(x, A, Γ), t, B)
    -------------------------
    ⊢⇐(Γ, λ(x, t), →(A, B))

```

### Translation to an algorithm.

#### 1. Terms as datatypes
We define datatypes according to the metavariable definitions.

#### 2. Metaterms as patterns
We assume that our language can pattern match on terms using metaterms. 
The operational semantics is clear (notice that
pattern matching does unification when two subterms are matched to the same 
variable).

### 3. Judgments as functions

A judgement `J(M1⁺, M2⁺, …, Mn⁺, N1⁻, N2⁻, …, Nm⁻)` is translated to a function 
`J : M1 × M2 × … × Mn → Maybe(N1 × N2 × … × Nm)`.
Then the inference rules are translated to pattern matching in the maybe monad.


By introducing auxiliary term constructors (tuples for each judgement), we can assume each
judgement has at most one input and at most one output. 

Assuming that `J` is declared as `J(M⁺, N⁻)`, 
we translate
```
J1(M1-mt, N1-mt), …, Jl(Ml-mt, Nl-mt)
-------------------------------------- (Rule)
J(M-mt, N-mt)
```

to:
    
```haskell
    J_Rule M = do
        M_mt  <- M
        N1_mt <- J1 M1_mt 
        …
        Nl_mt <- Jl Ml_mt 
        return N_mt
```

And then we define `J` as trying all the rules in the given order.
```haskell
    J M = J_Rule1 M <|> J_Rule2 M <|> … <|> J_RuleN M
```

## Example: Chipala's strict typing

The syntax is the same. 

```
        x ::= a() | b() | c() | ...
        X ::= A() | B() | C() | ...
        t ::= var(x⁺) | λ(x⁺, t⁺) | @(t⁺ , t⁺) | :(t⁺, T⁺)
        T ::= Var(X⁺) | ->(T⁺, T⁺)
        Γ ::= ·() | cons⇒(x⁺, T⁺, Γ⁺)
```

Let us use infix notation.

```
    Γ⁺ + Γ⁺ = Γ⁻

    ----------
    · + Γ = Γ

    Γ1 + Γ2 = Γ
    --------------------------------
    (x:T, Γ1) + Γ2 = (x:T, Γ)
```


```
    Γ⁺ = x⁺ : A⁻ + Γ⁻


    ----------------------
    (x:T, Γ) = x : T + Γ


    Γ = x:T + Γ'
    ----------------------------
    (y:T', Γ) = x:T + (y:T', Γ')
```

```
    Γ⁺ ⊢ t⁺ ⇒ T⁻ ⊣ Γ⁻


    -------------------------
    x:T, Γ ⊢⇒ var(x) ⇒ T ⊣ ·


    Γ ⊢ var(x) ⇒ T ⊣ ·
    -------------------------------------
    cons⇒(y, T', Γ), Γ' ⊢ var(x) ⇒ T ⊣ ·


    Γ ⊢ t1 ⇒ A → B ⊣ Γ1    Γ + Γ1 = Γ2     Γ2 ⊢ t2 ⇐ A ⊣ Γ3
    ---------------------------------------------------------
    Γ ⊢ t1 t2 ⇒ B ⊣ Γ3


    Γ ⊢ t ⇐ T ⊣ Γ'
    ----------------------
    Γ ⊢ (t:T) ⇒ T ⊣ Γ'


    Γ ⊢ t ⇒ B ⊣ Γ'   (Γ' = x:A + Γ'')
    ----------------------------------
    Γ ⊢ λx.t ⇒ A → B ⊣ Γ''
```

```
    Γ⁺ ⊢ t⁺ ⇐ T⁺ ⊣ Γ⁻


    Γ ⊢ t ⇒ T ⊣ Γ'
    --------------- (switch)
    Γ ⊢ t ⇐ T ⊢ Γ'


    x:A, Γ ⊢ t ⇐ B ⊣ Γ'
    -----------------------
    Γ ⊢⇐ λx.t ⇐ A → B ⊣ Γ'


    here it's important we've tried (switch) first
    -----------------------
    Γ ⊢⇐ var(x) ⇐ A ⊣ x:A
```







# DRAFT (ignore)


But what if a term has positive metavariables? They must be considered as outputs... as well as the metaterm itself
I guess we can consider a more abstract syntax:

        a(x⁺) | b(x⁺) | c(x⁺) |
        A(X⁺) | B(X⁺) | C(X⁺) |
        var(x⁻, t⁺) | λ(x⁻, t⁻, t⁺) | @(t⁻ , t⁻, t⁺) |
        Var(X⁻, T⁺) | ->(T⁻, T⁻, T⁺) |
        ·(Γ⁺) | cons⇒(x⁻, T⁻, Γ⁻, Γ⁺) | cons⇐(x⁻, T⁺, Γ⁻, Γ⁺)

Then the metaterms are circuits that have inputs and outputs. 

Then we have judgement forms
``
∊(x⁻, T⁻, Γ⁻)
``

together with inference rules, each of 

```
-------------------------
 ∊ (x, T, cons⇒(x, T, Γ))


 ∊ (x, T1, Γ)
-----------------------------
 ∊ (x, T1, cons⇒(y, T2, Γ))


-------------------------
 ∊ (x, T, cons⇐(x, T, Γ))


 ∊ (x, T1, Γ)
-----------------------------
 ∊ (x, T1, cons⇐(y, T2, Γ))

```






Then we have a grammar of terms, that are given as kinda  polynomial functors.
The left-hand side of ::= we call "metavariables", and their occurences in the
right-hand side are labeled as either ⁺ or ⁻, depending on whether they are interpreted as
inputs or outputs. 

Each functor definition has form M ::= T1(M11±, M12±, …, M1n1±) | T2(M21±, M22±, …, M2n2±) | … | Tn(Mm1±, Mm2±, …, Mmnm±)
where Ti are defined terminals, M.. are defined metavariables. We assume everting is type-consistent.

```
    x ::= a, b, c
    X ::= A, B, C
    t ::= var(x⁺) | λ(x⁺, T⁺, t⁺) | @(t⁺, t⁺)
    T ::= Var(X⁺) | ->(T⁺, T⁺)
    Γ ::= · | cons⇒(x⁺, T⁺, Γ⁺) | cons⇐(x⁺, T⁻, Γ⁺)
```

This functors define terms as an initial algebra. 
They also define metaterms---the terms that are not fully constructed
for instance: t, var(x), @(t1, t2), @(var(a), t). 
I don't know what it corresponds to categorically. Containers? Presheaves?


Finally, we have inference rules. Each rule defines a (directed) relation.
For example, 

```
∊(-, -, -)

-------------------------
 ∊ (x, T, cons⇒(x, T, Γ))


 ∊ (x, T1, Γ)
-----------------------------
 ∊ (x, T1, cons⇒(y, T2, Γ))


-------------------------
 ∊ (x, T, cons⇐(x, T, Γ))


 ∊ (x, T1, Γ)
-----------------------------
 ∊ (x, T1, cons⇐(y, T2, Γ))

```





```
⊢(-, -, +)

-------------------
 ⊢ (Var())
```
