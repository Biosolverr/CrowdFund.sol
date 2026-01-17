// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// @title CrowdFund - minimal crowdfunding contract (baseline)
/// @notice Baseline version intentionally keeps things simple (and improvable).
contract CrowdFund {
    struct Campaign {
        address creator;
        uint64 deadline;   // unix timestamp
        uint128 goal;      // wei
        uint128 pledged;   // wei
        bool claimed;
    }

    uint256 public campaignCount;
    mapping(uint256 => Campaign) public campaigns;
    mapping(uint256 => mapping(address => uint256)) public pledgedBy; // campaignId => user => amount

    function createCampaign(uint128 goal, uint64 durationSeconds) external returns (uint256 id) {
        require(goal > 0, "goal=0");
        require(durationSeconds >= 1 hours, "duration too short");

        id = ++campaignCount;
        campaigns[id] = Campaign({
            creator: msg.sender,
            deadline: uint64(block.timestamp) + durationSeconds,
            goal: goal,
            pledged: 0,
            claimed: false
        });
    }

    function pledge(uint256 id) external payable {
        Campaign storage c = campaigns[id];
        require(c.creator != address(0), "no campaign");
        require(block.timestamp < c.deadline, "ended");
        require(msg.value > 0, "value=0");

        pledgedBy[id][msg.sender] += msg.value;
        c.pledged += uint128(msg.value);
    }

    function unpledge(uint256 id, uint256 amount) external {
        Campaign storage c = campaigns[id];
        require(c.creator != address(0), "no campaign");
        require(block.timestamp < c.deadline, "ended");
        require(amount > 0, "amount=0");

        uint256 p = pledgedBy[id][msg.sender];
        require(p >= amount, "insufficient pledge");

        pledgedBy[id][msg.sender] = p - amount;
        c.pledged -= uint128(amount);

        // baseline: uses transfer (simple, but not always ideal)
        payable(msg.sender).transfer(amount);
    }

    function claim(uint256 id) external {
        Campaign storage c = campaigns[id];
        require(c.creator != address(0), "no campaign");
        require(msg.sender == c.creator, "not creator");
        require(block.timestamp >= c.deadline, "not ended");
        require(c.pledged >= c.goal, "goal not met");
        require(!c.claimed, "claimed");

        c.claimed = true;
        payable(c.creator).transfer(c.pledged);
    }

    function refund(uint256 id) external {
        Campaign storage c = campaigns[id];
        require(c.creator != address(0), "no campaign");
        require(block.timestamp >= c.deadline, "not ended");
        require(c.pledged < c.goal, "goal met");

        uint256 p = pledgedBy[id][msg.sender];
        require(p > 0, "nothing to refund");

        pledgedBy[id][msg.sender] = 0;
        // pledged field left as-is; refunds tracked via pledgedBy (simplifies baseline)
        payable(msg.sender).transfer(p);
    }

    function timeLeft(uint256 id) external view returns (uint256) {
        Campaign memory c = campaigns[id];
        if (c.creator == address(0) || block.timestamp >= c.deadline) return 0;
        return c.deadline - block.timestamp;
    }
}
