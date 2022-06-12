+++
title = "Crossword"
date = 2022-06-02
description = """
Crossword Problem solver
"""
+++

# Crossword-Problem
  
  
  The problem is trying to find legal words in k x k squares.Since one decision of character of a word will influence the next character and the word in rows and columns, the process of thinking about this problem is recursive, which means that I should write a recursive method to deal with this problem.When I think about this problem, I try to simplify how to find the word in the rows and columns. First, I think about how to use the character to determine the word on a board without small characters and blocks. When I determine the character of words, I will first check the dictionary that whether the word with this prefix exists for the rows and columns, then add another character and check the dictionary. If there is no prefix, then backtracking and delete the character and try another character. 
	
  For every square on the board, there are 26 characters that can put in the squares. Therefore, I use the template recursive method that for each character, test whether is valid in this square, if it is valid put it in the rows and columns array and solve next squares. Until the last square in the board is solved, the program will exit. For the small characters and block case, I used the switch statement to distinguish the method that uses for character and block. The basic idea of testing characters and blocks is the same. Only some of the operations are different from the empty squares. For the character, just test whether it is valid and add it to rows and columns without any loops. For the block part, I wrote a different type of isValid method to test whether the character before the block is a valid word. I also used the last index number for each row and column to track the last index of the block to help get the correct word.

  The whole logic and methods in the program are not so difficult, but the bugs and issues are very difficult to find. There are many edge cases and base cases in the code that I need to consider. For example, I need to consider the special location of characters, like in the first or last rows and columns, the location of the block, and the special version of the test. Try to include all the case is difficult, I need to test many files and print what happens in the file. Most of the files can get results by running the program. There is only one file that I cannot solve the problem, the testfile8c.txt. The testifle8c is very hard to find the bug since I do not know where the problem is, just pop up the wrong result ---no solution. I think the problem is just some case I did not realize if I can use some debugging tool to get every result of the input and output so that I think I can find where the bug and problem.
  
  ## Runtime for Different Files
  
  Filename|Runtime
  --------|-------
  test3a| 0.13s
  test3b| 0.12s
  test4a| 0.02s
  test4b| 0.04s
  test4c| 0.15s
  test4d| 0.002s
  test4e| 0.20s
  test4f| 0.38s
  test5a| 0.16s
  test6a| >30min
  test6b| 1009s
  test6c| >2h
  test7a|>2h
  test8a|>2h
  test8b|>2h
  test8c|>= 10min
  
  ## Asymptotic Analysis
 The runtime is exponential

Number of crossword locations in the puzzle = K Number of possible letters in a crossword location= 26 Number of words in the dictionary = N Number of characters in a word = M

Worst-Case runtime = branching factor^height * runtime of nonrecursive part = 26^(K)* (MN)
  