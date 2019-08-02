---
id: 28
title: Play tic tac toe with the help of advanced types in TypeScript
date: 2019-08-01T19:37:11+00:00
author: Barld Boot
layout: post
guid: http://barld.nl/?p=200
permalink: /200/tic-tac-toe/
videourl:
  - ""
categories:
  - Geen categorie
tags:
  - TypeScript
---
**In this article we are going to build the right types in TypeScript to play tic tac toe (in Dutch: boter, kaas en eieren) in a type safe way. We use the compiler to check if the sets are legal and to decide if there a player has won. There is no knowledge of advanced types expected but some basic understanding of types in programming and Typescript can be handy.**

## The tic tac toe board
A cell on the tic tac toe board can have three different states namely an `"X"`, an `"O"` or a blank cell (`" "`). In TypeScript we can create a union of string literals to define a type that support these cases. A string literal is a string limited to a specific string. A union is the choice between different types.

```ts
type Cases = "X" | "O" | " ";
```

In the case of our game, the value of a cell is limited to the above cases.

The board contains 9 separate cells which all have an identifier named from `A1` up on to `C3`
```ts
type Board = {
    A1: Cases, B1: Cases, C1: Cases,
    A2: Cases, B2: Cases, C2: Cases,
    A3: Cases, B3: Cases, C3: Cases,
};
```
This is the structure we are going to use to play our little game.

To create an empty playing board, we are going to use our first advanced concept. We are going to map our basic board to an empty board with a [mapped type](https://www.typescriptlang.org/docs/handbook/advanced-types.html#mapped-types).

```ts
type initialBoard = { [key in keyof Board]: " " };
```

We iterate through all keys of `Board` which is a union with all the cells `"A1" | "A2" | "A3" | ... | "C3"`, We ignore the value type of all cells and set it to empty cell (`" "`). This makes that we have an empty board.

## Finding empty cells
To ensure a fair game we should only allow sets on empty cell. Below you see the definition how to get all the empty cells as a union of string literals.

```ts
type AvailableCells<board extends Board> =
    { [key in keyof board]: board[key] extends " " ? key : never }[keyof board];
```
In the above example we have created a type alias with one generic parameter. The parameter `board` has the constrain that is should have at least the properties of a board defined before.

To explain the rest of the type we will reduce the code in steps. Imagine that our current board looks the following:

```ts
type board = { A1: "X", B1: " ", C1: " ", A2: "X", B2: "O", C2: " ", A3: "O", B3: " ", C3: " "}
```

The first step is that it is transformed to a new object with the help of the following rule

```ts
{ [key in keyof board]: board[key] extends " " ? key : never }
```

this will result in the following:

```ts
{ A1: never, B1: "B1", C1: "C1", A2: never, B2: never, C2: "C2", A3: never, B3: "B3", C3: "C3"}
```

It is iterating through all the fields in the object. With the help of a [conditional type](https://www.typescriptlang.org/docs/handbook/advanced-types.html#conditional-types) we check every field if it is empty as type so we will give the field name as type literal, every non empty field will be set to the type `never`. `never` is a special type in TypeScript for non existing types. It is used a lot for not reachable code.

In the next step the index operator is used to get all the value types of the object.
```ts
[keyof board]
```
This is working because the new object type has exact the same field names as the new created object. It says literally give me the value types of the following fields. Because the field names are known we could replace it in the following way

```ts
[keyof board] -> ["A1" | "A2" | "A3" | ... | "C3"]
```

This result in a union of all the value types in this case:

```ts
never | "B1" | "C1" | never | never | "C2" | never | "B3" | "C3"
```

In a union type all value types will be reduced to a unique set so the following is the same:

```ts
never | "B1" | "C1" | "C2" | "B3" | "C3"
```

And because `never` is a special case that stands for a not real existing value type it will be reduced away when there will be other value in the same value. This makes the following the finally reduced type which are the same as the empty cells:

```ts
"B1" | "C1" | "C2" | "B3" | "C3"
```

## Doing the set
Now we know which cells are free to do a set we can do the actually set. For this we create a type alias that accepts three parameters: the current board, which kind of sign you want to use (`"X" | "O"`) and in which cell you want to do this.

```ts
type DoSet<board extends Board, c extends "X" | "O", cell extends AvailableCells<board>> = {
    [key in keyof board]: key extends cell ? c : board[key]
};
```

To make sure that the cell chosen is an available cell we use the type we have created before `AvailableCells` as constrain on the cell chosen. We remap the board where everything is the same except for the cell chosen. This cell is changed to `"X"` or `"O"` depending on what is chosen. It returns the new changed type, but be aware that this is a new type, it is only possible to create new types it is impossible to mutate already existing types.

## Check if someone has won
The last thing we have to do before we can start playing is be able to check if there is already some player that has won. 
Because this is a small game we can check all possible winning combinations, this are 8 combinations. We can check every cell by using the index operator, for example:
```ts
board["A1"] // type: "X"
```

For a whole column we can do:
```ts
board["A1"] & board["A2"] & board["A3"] // type: "X" & "X" & " " 
```

Here we use an [intersection type](https://www.typescriptlang.org/docs/handbook/advanced-types.html#intersection-types). An intersection type is made by using the `&` operator. It says that the new type must have from both sides of the operator. Just like the union operator we can remove all elements that are not unique. For our above example it means we can reduce it to the following:
```ts
"X" & "X" & " " -> "X" & " "
```

If we had a winning combination it would look like the following:
```ts
"X" & "X" & "X" -> "X"
```

So if we want to know if `X` has won we just have to check if the type is `"X"`. For all combinations that looks like this:
```ts
type HasCWon<c extends "X" | "O", board extends Board> =
    c extends (
        // columns
        | (board["A1"] & board["A2"] & board["A3"])
        | (board["B1"] & board["B2"] & board["B3"])
        | (board["C1"] & board["C2"] & board["C3"])
        // rows
        | (board["A1"] & board["B1"] & board["C1"])
        | (board["A2"] & board["B2"] & board["C2"])
        | (board["A3"] & board["B3"] & board["C3"])
        // diagonal
        | (board["A1"] & board["B2"] & board["C3"])
        | (board["A3"] & board["B2"] & board["C1"])
    ) ? true : false;
```
If one of these combinations match with the char `c` we say that this char has won by returning `true` otherwise we return `false`. Make sure you use this order, when you change the order around of the extends it is not working correct because `"X" & "O" extends "X" -> true` and `"X" extends "X" & "O" -> false` what the behavior is we actually want.

The last step is to make a type that checks for both players if anyone has won. This looks the following:

```ts
type HasSomeOneWon<board extends Board> =
    HasCWon<"X", board> extends true ? "X has won"
    : HasCWon<"O", board> extends true ? "O has won"
    : "nobody has won";
```

We first check if `X` has won, so yes than we return `"X has won"` otherwise we check if `O` has won than we return `"O has won"`. When both cases are false we return nobody has won. 

Now we are ready to play

```ts
type set1 = DoSet<initialBoard, "X", "A1">;
type result1 = HasSomeOneWon<set1>; // "nobody has won"

type set2 = DoSet<set1, "O", "B2">;
type result2 = HasSomeOneWon<set2>; // "nobody has won"

type set3 = DoSet<set2, "X", "A2">;
type result3 = HasSomeOneWon<set3>; // "nobody has won"

type set4 = DoSet<set3, "O", "B3">;
type result4 = HasSomeOneWon<set4>; // "nobody has won"

type set5 = DoSet<set4, "X", "A3">;
type result5 = HasSomeOneWon<set5>; // "X has won"
```

## Conclude
In this blog i have introduced you to some of the advanced types in TypeScript and a way how you could use them. If you want to know better how to use the only advise I can give to you is to try it out and practice. 
Things you could try to make better what is not done yet is:

- Create a framework around this types so you could really use it (When it is now compiled to JavaScript you have only a blank file)
- track the state of the game an which player may do the next turn.

Last 5 months I have done my graduation internship at [Hoppinger](https://hoppinger.com) for my study [Informatica at Rotterdam University of Applied Sciences](https://www.hogeschoolrotterdam.nl/opleidingen/bachelor/informatica/voltijd/). My research was about using advanced types in TypeScript to validate programmer input for a configuration for a tool used intern at Hoppinger. Here I have learned a lot about the possibilities of advanced types in TypeScript but maybe even more about the shortcomings of the type system and the implementation in the TypeScript compiler. 
In September I will start my first "real" job by Hoppinger as Junior Software engineer.
