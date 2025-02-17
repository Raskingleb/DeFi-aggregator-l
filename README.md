// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/// @title A Simple Staking Contract for DeFi
/// @notice Users can deposit ERC20 tokens, earn rewards over time, and withdraw their stake along with rewards.
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
}

contract SimpleStaking {
    IERC20 public stakingToken;
    
    // Annual Percentage Yield (APY) in basis points.
    // For example, 1000 basis points = 10% APY.
    uint256 public constant APY_BASIS_POINTS = 1000;
    uint256 public constant SECONDS_IN_YEAR = 365 days;
    
    // Stake information for each user.
    struct Stake {
        uint256 amount;       // Total tokens staked by the user.
        uint256 rewardDebt;   // Accumulated rewards not yet claimed.
        uint256 lastUpdate;   // Timestamp of the last reward update.
    }
    
    mapping(address => Stake) public stakes;
    
    event Deposited(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    event RewardClaimed(address indexed user, uint256 reward);
    
    /// @notice Initializes the contract with the ERC20 token to be staked.
    /// @param _stakingToken Address of the ERC20 token contract.
    constructor(IERC20 _stakingToken) {
        stakingToken = _stakingToken;
    }
    
    /// @dev Internal function to update rewards for a user based on time elapsed.
    /// @param _user The address of the user.
    function updateReward(address _user) internal {
        Stake storage userStake = stakes[_user];
        if (userStake.amount > 0) {
            uint256 timeElapsed = block.timestamp - userStake.lastUpdate;
            // Calculate accrued reward:
            // reward = (staked amount * APY * timeElapsed) / (SECONDS_IN_YEAR * 10000)
            uint256 accruedReward = (userStake.amount * APY_BASIS_POINTS * timeElapsed) / (SECONDS_IN_YEAR * 10000);
            userStake.rewardDebt += accruedReward;
        }
        userStake.lastUpdate = block.timestamp;
    }
    
    /// @notice Deposit tokens into the staking contract.
    /// @param _amount The amount of tokens to stake.
    function deposit(uint256 _amount) external {
        require(_amount > 0, "Amount must be > 0");
        updateReward(msg.sender);
        
        // Transfer tokens from user to contract.
        require(stakingToken.transferFrom(msg.sender, address(this), _amount), "Token transfer failed");
        stakes[msg.sender].amount += _amount;
        emit Deposited(msg.sender, _amount);
    }
    
    /// @notice Withdraw staked tokens from the contract.
    /// @param _amount The amount of tokens to withdraw.
    function withdraw(uint256 _amount) external {
        Stake storage userStake = stakes[msg.sender];
        require(userStake.amount >= _amount, "Insufficient staked amount");
        updateReward(msg.sender);
        
        userStake.amount -= _amount;
        require(stakingToken.transfer(msg.sender, _amount), "Token transfer failed");
        emit Withdrawn(msg.sender, _amount);
    }
    
    /// @notice Claim the accumulated staking rewards.
    function claimReward() external {
        updateReward(msg.sender);
        uint256 reward = stakes[msg.sender].rewardDebt;
        require(reward > 0, "No rewards to claim");
        stakes[msg.sender].rewardDebt = 0;
        require(stakingToken.transfer(msg.sender, reward), "Reward transfer failed");
        emit RewardClaimed(msg.sender, reward);
    }
    
    /// @notice View function to see pending rewards for a user.
    /// @param _user The address of the user.
    /// @return Total pending rewards (accumulated + not yet updated).
    function pendingReward(address _user) external view returns (uint256) {
        Stake memory userStake = stakes[_user];
        uint256 timeElapsed = block.timestamp - userStake.lastUpdate;
        uint256 pending = (userStake.amount * APY_BASIS_POINTS * timeElapsed) / (SECONDS_IN_YEAR * 10000);
        return userStake.rewardDebt + pending;
    }
}
