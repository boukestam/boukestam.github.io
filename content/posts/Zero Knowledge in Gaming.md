+++ 
author = "Bouke Stam" 
title = "Zero Knowledge and Decentralized Gaming" 
date = "2022-07-11" 
description = "Thoughts on the usage of zero knowledge technologies in blockchain games" 
tags = [ "zero knowledge", "games" ] 
+++

Recent developments in zk-SNARK and zk-STARK technologies have made it much easier to implement zero knowledge applications. Vitalik recently did [a good post](https://vitalik.ca/general/2022/06/15/using_snarks.html) on the use cases in DeFi and governance. This post will focus on the use cases of zero knowledge in decentralized gaming.

### What is zero knowledge?
Every zero knowledge implementation starts with designing a circuit. This circuit contains some code that operates on private and public inputs. With a zk-SNARK we can proof that our private inputs successfully completed the circuit, without needing to reveal them. After the proof is created, it can be verified in a fraction of the time. This is especially useful in blockchain applications, where we are constrained by gas usage.

### Hiding your battleships
A good example where zero knowledge could be used is in 'Battleship'. In this game one person hides a fleet of ships and the other has to guess its location. The private input would be the location of the ships and the public input would be the guesses. The output whould then be which guesses successfully hit a ship. Every turn one player will guess, and send their guesses to the other player (or computer). They will then determine where the ships were hit and create a proof to show they are not lying.

![Battleship](/images/battleship.png)

### Hiding your crossword solution
Another use case would be competitive puzzle games. Let's say you wanted to make a crossword competition on the blockchain. Every morning you distribute the puzzle and the first person to solve it, wins a reward. The naive solution would require the winner to send his solution to a smart contract on the blockchain. But this message can be intercepted by opponents and duplicated with a higher gas fee so they run away with the reward.

In order to do this properly, we have to use a zero knowledge proof. We build a circuit that takes in the solution as a private input and only completes if the solution is correct. The winner can then create a proof and send this instead of his solution. By combining the proof with a signed message from the winner, we can ensure the proof is only usable by them.

### Sokoban puzzles
Zero knowledge can also be used when the solution is not known, or can be in many different configurations. For example in the sokoban puzzles in my upcoming game [Agnosis](https://twitter.com/AgnosisGames). This game revolves around pushing boxes around to get them in the right position. In this case you would need to build a circuit that takes in the scrambled puzzle and a private input with the steps you took to solve it. The circuit logic simulates the steps to see if it arrives at a valid solution. With this circuit you can proof that you solved the puzzle, without needing to reveal how.

![Agnosis](/images/agnosis.png)

### Space conquest
In the game [Dark Forest](https://zkga.me/) you start with a home planet and expand your territory by capturing new planets. Planets are hidden and can only be found by proving you know their location. Dark Forest implements a random noise function in a circuit, in order to generate planet locations without revealing them.

### The future is bright
Zero knowledge is only just starting to become known to the larger public. The technology is still new and under development, but starting to gain some serious traction. While zk-rollups are already being developed by multiple companies, zero knowledge gaming still has an untapped supply of new ideas and possibilities. Follow [Agnosis](https://twitter.com/AgnosisGames) on twitter to stay up to date on the latest updates and developments!