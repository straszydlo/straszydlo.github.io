---
layout: post
title: A story unfolds
category: Haskell
description: or how I tried to plant a tree
---
# A story unfolds

### or how I tried to plant a tree

I tried to re-implement the Unix `tree` tool in Haskell, as a learning exercise. The goal is to create a tool that, when provided with a directory, lists all the contents and prints them in a way similar to this:
```
$ tree
.
├── a
│   └── c
└── b
    ├── d
    └── e

5 directories, 0 files
```
It is well-defined, well-structured, and obviously recursive problem, so it should be easy to do, a no-brainer almost[^excuses].

I started hacking away at the tree.

The thing I would need first was a function to list directory contents, and another one to let me distinguish between directories and files. Quite fortunately, I had a pair of trusty axes on hand[^axes]:
```haskell
listDir :: File -> IO [File]
isDir :: File -> IO Boolean
```
A naïve implementation came to mind right away[^madeup]:
```haskell    
tree :: File -> IO ()
tree file = tree' "" file
  where tree' prefix file = do
          putStrLn (prefix ++ "- " ++ name file)
          isDirectory <- isDir file
          if isDirectory
          then do
            contents <- listDir file
            traverse_ (tree' (' ':prefix)) contents
          else return ()
```
It does not produce nice branch-looking lines between filenames, relying on indentation instead - but it is close enough.

Obviously, this code is clumsy - and not just because the utility functions are crude. No, this code is just clumsy and inflexible in itself - and what's even worse, it makes one cardinal sin.
_It mixes the responsibilities._

While it is not the gravest sin of them all, as one could argue that _traversing_ the filesystem and _printing_ the filenames are very strongly related, it would be nice to do these two separately. In fact, the added flexibility might come in handy if we want to do something more than just printing - and, as we're about to see, at pretty much no added cost.

So into the forest we go, preparing our tree-like data structure:
```haskell
data FileTree = RegularFile File | Directory File [FileTree]
```
Now we can write a function that grows these trees:
```haskell
buildTree :: File -> IO FileTree
buildTree file = do
  isDirectory <- isDir file
  if isDirectory
  then do
    contents <- listDir file
    subtrees <- traverse buildTree contents
    return (Directory file subtrees)
  else return (RegularFile file)
```
And we can restore the printing capability by writing another simple function:
```haskell
printTree :: FileTree -> String
printTree = unlines . printTree'
  where printTree' (RegularFile file) = ['-':(name file)]
        printTree' (Directory file contents) = ('-':(name file)):(subtrees contents)
        subtrees = map prepend . concatMap printTree'
        prepend path = ' ':path
```
Reasonably straightforward. We could even get away with making `FileTree` derive `Show` typeclass. The main benefit, though, is that we have separated the process of traversing the filesystem from the process of listing its contents - and should we ever need a function that goes into a directory and removes everything below a certain depth, we already have half of it implemented. Moreover, if we want to recover our initial function, we just need to compose a few blocks:
```haskell
tree :: File -> IO ()
tree file = do
  dirTree <- buildTree file
  let formatted = printTree dirTree
  putStrLn formatted
```
But can we go even further?

There seems to be nothing left to extract - we have separated traversal and printing, so what more could we possibly do? But there _is_ in fact a subtle thing. We are walking along the system paths as the trees unfold before us, but we do it in distinct steps: first we take a look around to gather all the possible paths; and then we walk along them.

Let's try to imagine a function like this. First, we need to be able to look around:
```haskell
type LookAround a = a -> [a]
```
Then, we need to be able to step along the way and construct subtrees - but this is simple recursion. We finally need to be able to construct our node (given the value to store inside the node and the paths to follow):
```haskell
type BuildNode t a = a -> [t a] -> t a
```
Having these, we can construct our tree builder:
```haskell
treeBuilder :: LookAround a -> BuildNode t a -> a -> t a
treeBuilder lookAround buildNode value = (buildNode value . childNodes . lookAround) value
  where childNodes = fmap (treeBuilder lookAround buildNode)
```
To be more confident that it works as expected, we can, for example, define these:
```haskell
data Tree a = Leaf a | Node a [Tree a] deriving Show

children :: LookAround Int
children 1 = []
children n = filter (\num -> n `mod` num == 0) [1..n-1]

nodes :: BuildNode Tree Int
nodes value [] = Leaf value
nodes value subtrees = Node value subtrees
```

and then run:
```haskell
print (treeBuilder children nodes 12)
```
to get a tree in which children of the node are the divisors of the value `v` stored in the node (smaller than `v` itself), `12` being the root.

```
Node 12 [
  Leaf 1,
  Node 2 [Leaf 1],
  Node 3 [Leaf 1],
  Node 4 [ 
    Leaf 1,
    Node 2 [Leaf 1]
  ],
  Node 6 [
    Leaf 1,
    Node 2 [Leaf 1],
    Node 3 [Leaf 1] 
  ]
]
```
It is neither useful nor spectacular, but gives us some reassurance. The only remaining problem is, how does `IO` fit the puzzle? Obviously, the `treeBuilder`, albeit nice, cannot handle the side effects.

So after another brief moment of hacking, I managed to stitch together an implementation:
```haskell
type LookAroundM m a = a -> m [a]
type BuildNodeM t m a = a -> [t a] -> m (t a)

treeBuilderM :: Monad m => LookAroundM m a -> BuildNodeM t m a -> a -> m (t a)
treeBuilderM lookAroundM buildNodeM value = lookAroundM value >>= childNodes >>= buildNodeM value
  where childNodes = traverse (treeBuilderM lookAroundM buildNodeM)
```
Now it contains the scary M-word, but it lets us redefine `treeBuilder` as a one-liner:
```haskell
treeBuilder lookAround buildNode value = runIdentity (treeBuilderM (return . lookAround) (\a b -> return (buildNode a b)) value)
```
The `lookAround` function now looks like this:
```haskell
lookAround :: LookAroundM IO File
lookAround file = do
  isDirectory <- isDir file
  if isDirectory
  then listDir file
  else return []
```
and the `buildNode` becomes:
```haskell
buildNode :: File -> [FileTree File] -> IO (FileTree File)
buildNode file subtrees = do
  isDirectory <- isDir file
  if isDirectory
  then return (Directory file subtrees)
  else return (RegularFile file)
```
with a minor modification to our `FileTree`:
```haskell
data FileTree a = RegularFile a | Directory a [FileTree a]
```
and to the `printTree` type signature:
```haskell
printTree :: FileTree File -> String
```
With all of this in place, we can rewrite our `buildTree` function:
```haskell
buildTree :: File -> IO (FileTree File)
buildTree = treeBuilderM lookAround buildNode
```
Wonderfully concise.

And thus, after a long walk among the trees, we arrive at a blissfully abstract, much less tangible function that traverses _any_ tree-like structure - as long as we are able to look around and see how it branches out. But is this useful in any way, or was it just an exercise in axe-swinging?

The benefit of separating listing and printing is quite obvious, because these are in fact completely unrelated tasks - but, to reiterate, it gives us more flexibility in how we want to do any of them (change the formatting without worrying about introducing listing-related bugs), more code reusability (as now we can use the filesystem traversal to, for example, create or delete files along certain paths), ease of testing (as we can test these in isolation) and reasoning (as there is less cognitive overhead when reading code that does _one_ thing instead of _two_ things tangled together). The wonderful nature of functions lets us compose the building blocks back together however we please, to either recover the original intent, or to introduce new functionality.

What we get from separating the tree traversal from listing new branches is, again, quite subtle. First, we know that our traversal works for every structure that at least resembles a tree - and it is ensured by Haskell type system. Second, finding stop condition-related bugs is now easier, as we shifted the responsibility for terminating the recursion to functions that _don't have to be recursive at all!_ Imagine how easy testing becomes!

The beauty of abstract constructs lies in their duality - they are incredibly rigid in one aspect, because their behavior is set in stone and virtually nothing can break it, and at the same time extremely flexible thanks to their generality. Once we create such a function, we can use it with confidence anywhere we wish (provided we can find a use case). The only downside is we need to spend more time to understand what is going on each time we encounter a new function - the more general it is, the harder to grasp it becomes. 

My `treeBuilderM` is, however, far from the most beautiful function. It can get better, prettier and even more general. The forest is thicker and spreads further, with many paths left to explore.

[^excuses]: I am by no means a Haskell expert, as I am still learning the language. Much of the code from this post could be written in a more elegant way, but I either don't know the proper constructs yet, or I am ignoring things like `ifM` (and even `$`) on purpose - to make the snippets more accessible to people from outside the Haskell world.

[^axes]:
    I am consciously ignoring the existence of symbolic links, along with the fact that a given file might not exist - for the sake of brevity.
    And here is the implementation, based on `listDirectory` and `doesDirectoryExist` from `System.Directory` module, from [`directory`](https://hackage.haskell.org/package/directory) package:
    ```haskell
    data File = File { base :: Maybe FilePath, name :: FilePath }

    fullPath :: File -> FilePath
    fullPath (File (Just base) name) = base ++ "/" ++ name
    fullPath (File Nothing name) = name

    listDir :: File -> IO [File]
    listDir file = do
      let dirPath = fullPath file
      contents <- listDirectory dirPath
      return (map (File (Just dirPath)) contents)

    isDir :: File -> IO Bool
    isDir = doesDirectoryExist . fullPath
    ```
    `FilePath` is an alias for `String`, defined in `System.Directory`.

    I decided to create the `File` type to avoid having to change the working directory, as the `listDirectory` function returns only the names of the files inside of a dir. It's not a proper implementation by any means, and it serves a purpose of simplifying the code a bit. It does also have a nasty side effect of making simple paths like `"."` into monstrosities like `File Nothing "."`.

    Also the code in this post imports `System.Directory`, `Data.Foldable` and `Data.Functor.Identity`.
[^madeup]: I made this part up, I actually started with the `data Tree` implementation.

