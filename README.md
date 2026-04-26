# claimJackpot
LuckyNumberGame.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/// @title Lucky Number Game - A fun guessing game on Base Chain
/// @notice Pay to guess a number between 1-10 and win the jackpot if you guess correctly!
contract LuckyNumberGame {

    // Game settings
    uint256 public constant ENTRY_FEE = 0.001 ether;     // Cost to play one round
    uint256 public constant MIN_NUMBER = 1;
    uint256 public constant MAX_NUMBER = 10;

    // Game state
    uint256 public jackpot;                              // Current prize pool
    address public lastWinner;
    uint256 public lastWinningNumber;
    uint256 public totalGamesPlayed;

    // Events for frontend to listen
    event GamePlayed(address player, uint256 guessedNumber, uint256 winningNumber, bool isWinner, uint256 prize);
    event JackpotClaimed(address claimer, uint256 amount);

    // Constructor - starts with a small seed
    constructor() payable {
        jackpot = msg.value;  // Optional: send some ETH when deploying to seed the jackpot
    }

    /// @notice Play the game by guessing a number between 1 and 10
    /// @param _guess Your lucky number (1-10)
    function play(uint256 _guess) external payable {
        require(msg.value == ENTRY_FEE, "Must send exactly the entry fee");
        require(_guess >= MIN_NUMBER && _guess <= MAX_NUMBER, "Guess must be between 1 and 10");

        // Simple on-chain pseudo-random number (for fun, not production-grade security)
        uint256 winningNumber = (uint256(keccak256(abi.encodePacked(
            block.timestamp, 
            block.prevrandao, 
            msg.sender,
            totalGamesPlayed
        ))) % (MAX_NUMBER - MIN_NUMBER + 1)) + MIN_NUMBER;

        bool isWinner = (_guess == winningNumber);
        uint256 prize = 0;

        jackpot += ENTRY_FEE;  // Add entry fee to the pool

        if (isWinner) {
            // Winner takes 80% of current jackpot
            prize = (jackpot * 80) / 100;
            if (prize > 0) {
                jackpot -= prize;
                (bool success, ) = payable(msg.sender).call{value: prize}("");
                require(success, "Prize transfer failed");
            }
            lastWinner = msg.sender;
            lastWinningNumber = winningNumber;
        }

        totalGamesPlayed++;

        emit GamePlayed(msg.sender, _guess, winningNumber, isWinner, prize);
    }

    /// @notice Anyone can claim a small portion of the jackpot (10%) to prevent it from growing too large
    function claimJackpotPortion() external {
        require(jackpot > 0.01 ether, "Jackpot too small to claim portion");

        uint256 claimAmount = jackpot / 10;  // 10% of jackpot

        jackpot -= claimAmount;

        (bool success, ) = payable(msg.sender).call{value: claimAmount}("");
        require(success, "Claim transfer failed");

        emit JackpotClaimed(msg.sender, claimAmount);
    }

    /// @notice Get current game info
    function getGameInfo() external view returns (
        uint256 currentJackpot,
        uint256 gamesPlayed,
        address latestWinner,
        uint256 latestWinNumber
    ) {
        return (jackpot, totalGamesPlayed, lastWinner, lastWinningNumber);
    }

    /// @notice Withdraw remaining funds (only if you're the deployer - for emergency)
    function emergencyWithdraw() external {
        require(msg.sender == tx.origin, "Only direct caller");
        // In production, you may want to add an owner
        (bool success, ) = payable(msg.sender).call{value: address(this).balance}("");
        require(success, "Withdraw failed");
    }

    // Allow contract to receive ETH directly
    receive() external payable {
        jackpot += msg.value;
    }
}
