# revoke
revokes set tokens with set timrt

real contracts starts from line 5


// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract TokenRevoke {
    // Reference to the ERC20 token contract (hardcoded)
    IERC20 public token = IERC20(0xf817257fed379853cDe0fa4F97AB987181B1E5Ea);

    // Struct to store transfer details
    struct Transfer {
        address sender;
        address recipient;
        uint256 amount;
        uint256 deadline;
        bool claimed;
    }

    // Mapping to store transfers by a unique transfer ID
    mapping(bytes32 => Transfer) public transfers;

    // Events for tracking actions
    event TokensSent(bytes32 indexed transferId, address indexed sender, address indexed recipient, uint256 amount, uint256 deadline);
    event TokensClaimed(bytes32 indexed transferId, address indexed recipient, uint256 amount);
    event TokensRevoked(bytes32 indexed transferId, address indexed sender, uint256 amount);

    // Function to send tokens with a 24-hour revocation period
    function sendTokens(address _recipient, uint256 _amount) external returns (bytes32) {
        require(_recipient != address(0), "Invalid recipient address");
        require(_amount > 0, "Amount must be greater than 0");
        require(token.balanceOf(msg.sender) >= _amount, "Insufficient token balance");
        require(token.allowance(msg.sender, address(this)) >= _amount, "Insufficient allowance");

        // Generate a unique transfer ID
        bytes32 transferId = keccak256(abi.encodePacked(msg.sender, _recipient, _amount, block.timestamp));

        // Calculate deadline (24 hours from now)
        uint256 deadline = block.timestamp + 24 hours;

        // Store transfer details
        transfers[transferId] = Transfer({
            sender: msg.sender,
            recipient: _recipient,
            amount: _amount,
            deadline: deadline,
            claimed: false
        });

        // Transfer tokens to this contract
        require(token.transferFrom(msg.sender, address(this), _amount), "Token transfer failed");

        // Emit event
        emit TokensSent(transferId, msg.sender, _recipient, _amount, deadline);

        return transferId;
    }

    // Function for the recipient to claim tokens
    function claimTokens(bytes32 _transferId) external {
        Transfer storage transfer = transfers[_transferId];
        require(transfer.recipient == msg.sender, "Not the recipient");
        require(!transfer.claimed, "Tokens already claimed");
        require(block.timestamp <= transfer.deadline, "Claim period has expired");
        require(transfer.amount > 0, "No tokens to claim");

        // Mark as claimed
        transfer.claimed = true;

        // Transfer tokens to the recipient
        require(token.transfer(transfer.recipient, transfer.amount), "Token transfer failed");

        // Emit event
        emit TokensClaimed(_transferId, transfer.recipient, transfer.amount);
    }

    // Function for the sender to revoke tokens after the deadline
    function revokeTokens(bytes32 _transferId) external {
        Transfer storage transfer = transfers[_transferId];
        require(transfer.sender == msg.sender, "Not the sender");
        require(!transfer.claimed, "Tokens already claimed");
        require(block.timestamp > transfer.deadline, "Revocation period not started");
        require(transfer.amount > 0, "No tokens to revoke");

        // Get the amount to revoke
        uint256 amount = transfer.amount;

        // Reset the transfer
        transfer.amount = 0;
        transfer.claimed = true; // Prevent further actions

        // Transfer tokens back to the sender
        require(token.transfer(transfer.sender, amount), "Token transfer failed");

        // Emit event
        emit TokensRevoked(_transferId, transfer.sender, amount);
    }

    // Function to check transfer details
    function getTransferDetails(bytes32 _transferId)
        external
        view
        returns (
            address sender,
            address recipient,
            uint256 amount,
            uint256 deadline,
            bool claimed
        )
    {
        Transfer storage transfer = transfers[_transferId];
        return (
            transfer.sender,
            transfer.recipient,
            transfer.amount,
            transfer.deadline,
            transfer.claimed
        );
    }
}
