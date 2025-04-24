/**
 *Submitted for verification at BscScan.com on 2025-04-23
*/

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.29;
/*
 *  █████╗ ██╗ ██████╗ ███╗   ██╗ ██████╗    ███████╗██╗   ██╗███╗   ██╗██████╗ 
 * ██╔══██╗██║██╔═══██╗████╗  ██║██╔════╝    ██╔════╝██║   ██║████╗  ██║██╔══██╗
 * ███████║██║██║   ██║██╔██╗ ██║██║         █████╗  ██║   ██║██╔██╗ ██║██║  ██║
 * ██╔══██║██║██║   ██║██║╚██╗██║██║         ██╔══╝  ██║   ██║██║╚██╗██║██║  ██║
 * ██║  ██║██║╚██████╔╝██║ ╚████║╚██████╗ █  ██║     ╚██████╔╝██║ ╚████║██████╔╝
 * ╚═╝  ╚═╝╚═╝ ╚═════╝ ╚═╝  ╚═══╝ ╚═════╝    ╚═╝      ╚═════╝ ╚═╝  ╚═══╝╚═════╝ 
 * 
 * aionic.fund - Decentralized Investment Platform
 */
interface IERC20 {
    function transfer(address recipient, uint256 amount) external returns (bool);
}

contract AionicfundToken {
    string public name = "Aionicfund Token";
    string public symbol = "AION";
    uint8 public decimals = 18;
    uint256 public totalSupply;
    uint256 public constant INITIAL_SUPPLY = 1_000_000_000 * 10**18;
    uint256 public constant MIN_SUPPLY = 20_000_000 * 10**18;

    address public owner;
    address public pendingOwner;
    bool public paused;
    bool private locked;

    uint256 public constant MAX_HOLDERS = 5000;
    uint256 public constant MIN_HOLDER_BALANCE = 100 * 10 ** 18;
    uint256 public constant BURN_FEE = 5;
    uint256 public constant OWNER_FEE = 5;
    uint256 public constant HOLDER_FEE = 15;
    uint256 public constant FEE_DIVISOR = 1000;
    uint256 public rewardPool;
    uint256 public stakingRewardPool;
    uint256 public lastRewardDistribution;
   
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    mapping(address => uint256) public stakingBalance;
    mapping(address => uint256) public stakingStartTime;
    mapping(address => uint256) public holderBalances;
    mapping(address => uint256) public holderIndices;
   
   
    mapping(address => bool) public isHolder;
    mapping(address => bool) public isWhitelisted;
    address[] public activeHolders;
    uint256 public lastDistributedIndex;
    
    struct Grant {
        uint256 totalAmount;
        uint256 claimedAmount;
        uint256 cliff;
        uint256 duration;
        uint256 interval;
    }
    mapping(address => Grant) public grants;
    address public vestingContract;

    // Errors
    error InsufficientBalance(uint256 available, uint256 required);
    error ExceedsMaxLimit(uint256 maxAllowed);
    error AddressLocked(address account);

    // Events
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event RewardDistributed(address indexed holder, uint256 amount);
    event HolderAdded(address indexed holder);
    event HolderRemoved(address indexed holder);
    event Burned(address indexed holder, uint256 amount);
    event Staked(address indexed user, uint256 amount);
    event Unstaked(address indexed user, uint256 amount);
    event RewardClaimed(address indexed beneficiary, uint256 amount);
    event VestingGrantAdded(address indexed beneficiary, uint256 amount);
    event VestingClaimed(address indexed beneficiary, uint256 amount);
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    modifier whenNotPaused() {
        require(!paused, "Contract is paused");
        _;
    }

    modifier nonReentrant() {
        require(!locked, "Reentrant call");
        locked = true;
        _;
        locked = false;
    }
    function _transfer(address from, address to, uint256 value) internal {
    require(balanceOf[from] >= value, "Insufficient balance");
    require(to != address(0), "Invalid address");
    
    balanceOf[from] -= value;
    balanceOf[to] += value;
    
    emit Transfer(from, to, value);
}

function _transferWithFrom(address _from, address _to, uint256 _value) internal {
    require(balanceOf[_from] >= _value, "Balance too low");

    uint256 burnAmt;
    uint256 ownerFee;
    uint256 holderFee;
    uint256 totalFee;

    if (!isWhitelisted[_from]) {
        (burnAmt, ownerFee, holderFee) = _calculateFees(_value, _from);
        totalFee = burnAmt + ownerFee + holderFee;
    }

    uint256 net = _value - totalFee;
    balanceOf[_from] -= _value;
    balanceOf[_to] += net;

    if (burnAmt > 0 && totalSupply > MIN_SUPPLY) {
        uint256 actualBurn = (totalSupply - burnAmt >= MIN_SUPPLY) ? burnAmt : totalSupply - MIN_SUPPLY;
        totalSupply -= actualBurn;
        emit Burned(_from, actualBurn);
        emit Transfer(_from, address(0), actualBurn);
    }

    if (ownerFee > 0) {
        balanceOf[owner] += ownerFee;
        emit Transfer(_from, owner, ownerFee);
    }

    if (holderFee > 0) {
        rewardPool += holderFee;
        emit Transfer(_from, address(this), holderFee);
    }

    emit Transfer(_from, _to, net);
    _updateHolders(_from);
    _updateHolders(_to);
}

    constructor() {
        owner = msg.sender;
        totalSupply = 1_000_000_000 * 10 ** uint256(decimals);
        balanceOf[msg.sender] = totalSupply;
    }

    
    function approve(address spender, uint256 amount) public returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transfer(address to, uint256 value) public whenNotPaused nonReentrant returns (bool success) {
    if (balanceOf[msg.sender] < value) revert InsufficientBalance(balanceOf[msg.sender], value);
    require(to != address(0), "Invalid address");
    require(to != address(this), "Cannot transfer to contract");

    (uint256 burnAmount, uint256 ownerAmount, uint256 holderAmount) = _calculateFees(value, msg.sender); // پارامتر msg.sender اضافه شد
    uint256 netAmount = value - burnAmount - ownerAmount - holderAmount;

    balanceOf[msg.sender] -= value;
    balanceOf[to] += netAmount;
    balanceOf[owner] += ownerAmount;
    rewardPool += holderAmount;
    totalSupply -= burnAmount;

    _updateHolders(to);
    emit Transfer(msg.sender, to, netAmount);
    return true;
}

    function _calculateFees(uint256 amount, address from) internal view returns (uint256, uint256, uint256) {
       if (from == owner || isWhitelisted[from]) {
           return (0, 0, 0);
       }
       return (
           (amount * BURN_FEE) / FEE_DIVISOR,
           (amount * OWNER_FEE) / FEE_DIVISOR,
           (amount * HOLDER_FEE) / FEE_DIVISOR
       );
   }
     function transferFrom(address _from, address _to, uint256 _value) public returns (bool) {
        require(_to != address(0), "Zero address");
        require(_value <= allowance[_from][msg.sender], "Allowance exceeded");
        allowance[_from][msg.sender] -= _value;
        _transferWithFrom(_from, _to, _value);
        return true;
    }

  
    function _updateHolders(address user) internal {
        if (!isHolder[user]) {
            if (activeHolders.length >= MAX_HOLDERS) revert ExceedsMaxLimit(MAX_HOLDERS);
            activeHolders.push(user);
            holderIndices[user] = activeHolders.length;
            isHolder[user] = true;
            emit HolderAdded(user);
        }
        holderBalances[user] = balanceOf[user];

        if (balanceOf[user] < MIN_HOLDER_BALANCE) {
            _removeHolder(user);
        }
    }

    function _removeHolder(address user) private {
        uint256 indexToRemove = holderIndices[user] - 1;
        if (indexToRemove != activeHolders.length - 1) {
            address lastHolder = activeHolders[activeHolders.length - 1];
            activeHolders[indexToRemove] = lastHolder;
            holderIndices[lastHolder] = indexToRemove + 1;
        }
        activeHolders.pop();
        delete holderIndices[user];
        delete holderBalances[user];
        delete isHolder[user];
        emit HolderRemoved(user);
    }

    function distributeHolderRewards(uint256 maxIterations) external onlyOwner nonReentrant {
    require(rewardPool > 0, "No rewards to distribute");
    
    uint256 initialRewardPool = rewardPool;
    uint256 eligibleTotal = calculateEligibleTotal();
    require(eligibleTotal > 0, "No eligible holders");
    
    uint256 distributedAmount;
    uint256 iterations;
    
    while (iterations < maxIterations && rewardPool > 0 && lastDistributedIndex < activeHolders.length) {
        address holder = activeHolders[lastDistributedIndex];
        
        if (balanceOf[holder] >= MIN_HOLDER_BALANCE) {
            uint256 reward = (initialRewardPool * balanceOf[holder]) / eligibleTotal;
            
            if (reward > 0) {
                balanceOf[holder] += reward;
                distributedAmount += reward;
                rewardPool -= reward;
                emit RewardDistributed(holder, reward);
            }
        }
        
        lastDistributedIndex++;
        iterations++;
        
        if (lastDistributedIndex >= activeHolders.length) {
            lastDistributedIndex = 0;
            break;
        }
    }
}

     function stake(uint256 amount) public {
        require(amount >= 100 * 10**decimals, "Minimum stake is 100 AION");
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");

        balanceOf[msg.sender] -= amount;
        stakingBalance[msg.sender] += amount;
        stakingStartTime[msg.sender] = block.timestamp;

        _updateHolders(msg.sender);
        emit Staked(msg.sender, amount);
    }

    function claimStakingReward() public nonReentrant {
        require(stakingBalance[msg.sender] > 0, "No stake found");
        require(block.timestamp >= stakingStartTime[msg.sender] + 7 days, "Stake time not passed");

        uint256 stakingDuration = (block.timestamp - stakingStartTime[msg.sender]) / 1 days;
        uint256 rate = stakingBalance[msg.sender] >= 1_000_000 * 10**decimals ? 10 : 8;
        
        
        if (stakingDuration >= 30 days) rate += 2;
        if (stakingDuration >= 90 days) rate += 3;
        if (stakingDuration >= 180 days) rate += 5;

        uint256 reward = (stakingBalance[msg.sender] * rate * stakingDuration) / (365 days * 100);

        if (reward > stakingRewardPool) reward = stakingRewardPool;

        stakingRewardPool -= reward;
        balanceOf[msg.sender] += reward;
        stakingStartTime[msg.sender] = block.timestamp;

        emit RewardClaimed(msg.sender, reward);
    }

    function unstake() public {
        uint256 amount = stakingBalance[msg.sender];
        require(amount > 0, "No stake to unstake");

        stakingBalance[msg.sender] = 0;
        balanceOf[msg.sender] += amount;

        _updateHolders(msg.sender);
        emit Unstaked(msg.sender, amount);
    }

    function fundStakingRewardPool(uint256 amount) external onlyOwner {
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        balanceOf[msg.sender] -= amount;
        stakingRewardPool += amount;
    }


    function calculateEligibleTotal() private view returns (uint256 total) {
        for (uint256 i = 0; i < activeHolders.length; i++) {
            if (balanceOf[activeHolders[i]] >= MIN_HOLDER_BALANCE) {
                total += balanceOf[activeHolders[i]];
            }
        }
        return total;
    }

    function getActiveHoldersCount() external view returns (uint256) {
        return activeHolders.length;
    }

    function getHolderRewardEstimate(address holder) external view returns (uint256) {
        if (balanceOf[holder] < MIN_HOLDER_BALANCE || rewardPool == 0) return 0;
        uint256 eligibleTotal = calculateEligibleTotal();
        return (rewardPool * balanceOf[holder]) / eligibleTotal;
    }

    function pause() external onlyOwner {
        paused = true;
    }

    function unpause() external onlyOwner {
        paused = false;
    }

    function rescueToken(address tokenAddress, uint256 amount) external onlyOwner {
        IERC20(tokenAddress).transfer(owner, amount);
    }

    function setVestingContract(address vestingAddr) external onlyOwner {
        require(vestingAddr != address(0), "Zero address");
        vestingContract = vestingAddr;
    }

    function getVestingInfo(address beneficiary) public view returns (
        uint256 totalAmount,
        uint256 claimedAmount,
        uint256 availableAmount,
        uint256 nextClaimTime
    ) {
        Grant memory grant = grants[beneficiary];
        uint256 elapsed = block.timestamp > grant.cliff ? block.timestamp - grant.cliff : 0;
        uint256 totalPeriods = grant.duration / grant.interval;
        uint256 periodsElapsed = elapsed / grant.interval;
        if (periodsElapsed > totalPeriods) periodsElapsed = totalPeriods;

        uint256 totalClaimable = (grant.totalAmount * periodsElapsed) / totalPeriods;
        uint256 claimableNow = totalClaimable > grant.claimedAmount ? totalClaimable - grant.claimedAmount : 0;
        uint256 nextClaim = grant.cliff + ((periodsElapsed + 1) * grant.interval);

        return (
            grant.totalAmount,
            grant.claimedAmount,
            claimableNow,
            nextClaim
        );
    }
    function whitelistAddress(address user, bool status) external onlyOwner {
        isWhitelisted[user] = status;
    }
   
    function acceptOwnership() external {
        require(msg.sender == pendingOwner, "Not pending owner");
        emit OwnershipTransferred(owner, pendingOwner);
        owner = pendingOwner;
        pendingOwner = address(0);
    }
}