```solidity
// Minimal snippet (README): core storage + createCampaign

struct Campaign {
    address creator;
    uint64 deadline;
    uint128 goal;
    uint128 pledged;
    bool claimed;
}

uint256 public campaignCount;
mapping(uint256 => Campaign) public campaigns;

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

    
