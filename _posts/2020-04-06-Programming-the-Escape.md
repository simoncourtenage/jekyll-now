---
layout: post
title: Programming the Escape
---

Stuck inside, during this period of coronoavirus lockdown, and have nothing to do?  You could do some programming challenges or read that
philosophy text you've been putting off for years - or both.

I came across the Coding19 [website](https://coding19-imdea.github.io) via twitter, which sets daily coding challenges for people to try.  [Today's challenge](https://coding19-imdea.github.io/problems/problem10.html) was about helping three burglars to escape with their loot of
diamonds, gold and cash.

For example, if burglar 1 has 100 diamonds, 50 gold, 10 cash, and burglar 2 has 50 diamonds, 100 gold, 10 cash, and lastly, burglar 3 has 10 diamonds, 50 gold, and 100 cash, then the items move between the burglars as follows:

Burglar 1: 100 diamonds (which they already have) + 50 (from B2) + 10 (from B3) = 60 moves
Burglar 2: 100 gold (already have) + 50 (from B2) + 50 (from B3) = 100 moves
Burglar 3: 100 cash (already have) + 10 (from B1) + 10 (from b2) = 20 moves

So total moves = 60 + 100 + 20 = 180

Below is my solution, written in Haskell.  I abstracted the problem into summing a list of 3 lists of items, where each item is an identifier and a value.  The first list is the list of diamonds among the 3 burglars, the second the gold, and the third the cash.  Each list is then summed, excluding the item with the maximum value.  There is an execption to this, which is where the max item's user has already been allocated a sum from a previous sublist.  Some assumptions are made - mostly to do with there always being three sublists.

```
module Main where

import Data.List (sortBy)

main :: IO ()
main = do
    line <- getLine
    let ns = (convert (words line) :: [Int])
    let moves = mover ns
    putStrLn (show moves)


convert :: Read a => [String] -> [a]
convert = map read

-- data structures and types

type Id = Int
type Item = (Id,Int)
type Collection = [[Item]]


-- top-level function for moving items
mover :: [Int] -> Int
mover ns = mover' [] 0 sortedItems
    where sortedItems = (sorter . parseItems) ns
          -- firstId = fst . head . head $ sortedItems

-- parse the numbers into a list of lists.  Each sub-list represents the values
-- of one type of item.  So the first sublist will be all the diamonds, the second
-- all the gold, and the last all the cash values.  Moreover, each item is identified
-- with a particular user.  Hence each item is a tuple of id and value.
parseItems :: [Int] -> Collection
parseItems = parseItems' [[],[],[]] 0

parseItems' :: Collection -> Id -> [Int] -> Collection
parseItems' col _ [] = col
parseItems' [d,g,c] id (n1:n2:n3:ns) = parseItems' [(id,n1):d,(id,n2):g,(id,n3):c] (id+1) ns


-- sort each sublist into descending order of item value, then sort the sublists into descending order
-- of max value
sorter :: Collection -> Collection
sorter [d,g,c] = reverse $ sortBy (\a b -> compare ((snd . head) a) ((snd . head) b)) [itemSorter d,itemSorter g,itemSorter c]

itemSorter :: [Item] -> [Item]
itemSorter l = reverse $ sortBy (\a b -> compare (snd a) (snd b)) l

-- calculation of moves is based on the idea that the one item we don't move is the max value in
-- each list of items, unless the user with that max has already had another type of item transferred
-- to him.  So for diamonds, the user with the max value gets all the diamonds.  Then the user with the
-- max gold gets all the gold given to him unless he already has all the diamonds etc.
mover' :: [Id] -> Int -> Collection -> Int
mover' ids mvs cs | length cs == length ids = mvs
                  | otherwise             = mover' (i:ids) (mvs + m) cs
    where pos = length ids
          (i,m) = moves (cs !! pos) ids

-- calculate moves for one list of items
moves :: [Item] -> [Id] -> (Id,Int)
moves l ids = (id,(adder . fst) pair + (adder . snd) pair)
    where pair = splitWhen (pred ids) l
          adder = sum . (map snd)
          pred = (\list (i,v) -> elem i list)
          id = (fst . head) $ dropWhile (pred ids) l

-- this function works out which values to move, by excluding the unallocated user
-- who has max item value
splitWhen :: (a -> Bool) -> [a] -> ([a],[a])
splitWhen p l = (prefix, tail suffix)
    where
    prefix = takeWhile p l
    suffix = drop (length prefix) l

