// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract GamingItemTrading {
    
    // Item rarity levels
    enum Rarity { Common, Rare, Epic, Legendary, Mythic }
    
    // Item categories
    enum Category { Weapon, Armor, Consumable, Accessory, Material }
    
    // Trade status
    enum TradeStatus { Open, Completed, Cancelled }

    // Item structure
    struct GameItem {
        string name;
        Category category;
        Rarity rarity;
        uint256 level;
        uint256 attack;
        uint256 defense;
        uint256 createdAt;
        address creator;
        address owner;
        bool exists;
    }

    // Trade structure
    struct Trade {
        uint256 tradeId;
        address seller;
        uint256 tokenId;
        uint256 price;
        TradeStatus status;
        uint256 createdAt;
        uint256 expiresAt;
    }

    // State variables
    uint256 private _tokenIdCounter;
    uint256 private _tradeIdCounter;
    address public owner;
    
    // Mappings
    mapping(uint256 => GameItem) public gameItems;
    mapping(uint256 => Trade) public trades;
    mapping(address => uint256[]) public userItems;
    mapping(Rarity => uint256) public rarityMultiplier;
    
    // Events
    event ItemCreated(uint256 indexed tokenId, address indexed creator, string name, Rarity rarity);
    event TradeCreated(uint256 indexed tradeId, address indexed seller, uint256 indexed tokenId, uint256 price);
    event TradeCompleted(uint256 indexed tradeId, address indexed buyer, uint256 price);
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);

    // Modifiers
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }

    modifier nonReentrant() {
        _;
    }

    constructor() {
        owner = msg.sender;
        // Set rarity multipliers for pricing
        rarityMultiplier[Rarity.Common] = 1;
        rarityMultiplier[Rarity.Rare] = 3;
        rarityMultiplier[Rarity.Epic] = 10;
        rarityMultiplier[Rarity.Legendary] = 30;
        rarityMultiplier[Rarity.Mythic] = 100;
    }

    /**
     * @dev Core Function 1: Create a new game item NFT
     * @param name Name of the item
     * @param category Category of the item (0-4)
     * @param rarity Rarity of the item (0-4)
     * @param attack Attack value of the item
     * @param defense Defense value of the item
     */
    function createItem(
        string memory name,
        Category category,
        Rarity rarity,
        uint256 attack,
        uint256 defense
    ) public returns (uint256) {
        uint256 tokenId = _tokenIdCounter;
        _tokenIdCounter++;

        gameItems[tokenId] = GameItem({
            name: name,
            category: category,
            rarity: rarity,
            level: 1,
            attack: attack,
            defense: defense,
            createdAt: block.timestamp,
            creator: msg.sender,
            owner: msg.sender,
            exists: true
        });

        userItems[msg.sender].push(tokenId);

        emit ItemCreated(tokenId, msg.sender, name, rarity);
        emit Transfer(address(0), msg.sender, tokenId);
        return tokenId;
    }

    /**
     * @dev Core Function 2: List an item for trading
     * @param tokenId ID of the token to trade
     * @param price Price in wei for the item
     * @param duration Duration in seconds for the trade to remain active
     */
    function listItemForTrade(
        uint256 tokenId,
        uint256 price,
        uint256 duration
    ) public {
        require(gameItems[tokenId].exists, "Token does not exist");
        require(gameItems[tokenId].owner == msg.sender, "Not the owner of this item");
        require(price > 0, "Price must be greater than 0");
        require(duration > 0, "Duration must be greater than 0");

        uint256 tradeId = _tradeIdCounter;
        _tradeIdCounter++;

        trades[tradeId] = Trade({
            tradeId: tradeId,
            seller: msg.sender,
            tokenId: tokenId,
            price: price,
            status: TradeStatus.Open,
            createdAt: block.timestamp,
            expiresAt: block.timestamp + duration
        });

        // Transfer ownership to contract for escrow
        gameItems[tokenId].owner = address(this);
        _removeFromUserItems(msg.sender, tokenId);

        emit TradeCreated(tradeId, msg.sender, tokenId, price);
    }

    /**
     * @dev Core Function 3: Execute a trade by purchasing an item
     * @param tradeId ID of the trade to execute
     */
    function executeTrade(uint256 tradeId) public payable nonReentrant {
        Trade storage trade = trades[tradeId];
        
        require(trade.status == TradeStatus.Open, "Trade is not open");
        require(block.timestamp <= trade.expiresAt, "Trade has expired");
        require(msg.value >= trade.price, "Insufficient payment");
        require(msg.sender != trade.seller, "Cannot buy your own item");

        trade.status = TradeStatus.Completed;

        // Transfer item ownership to buyer
        gameItems[trade.tokenId].owner = msg.sender;
        
        // Update user items mappings
        userItems[msg.sender].push(trade.tokenId);

        // Calculate platform fee (2.5%)
        uint256 platformFee = (trade.price * 25) / 1000;
        uint256 sellerAmount = trade.price - platformFee;

        // Pay seller
        payable(trade.seller).transfer(sellerAmount);
        
        // Refund excess payment to buyer
        if (msg.value > trade.price) {
            payable(msg.sender).transfer(msg.value - trade.price);
        }

        emit TradeCompleted(tradeId, msg.sender, trade.price);
        emit Transfer(trade.seller, msg.sender, trade.tokenId);
    }

    /**
     * @dev Get details of a specific game item
     * @param tokenId ID of the token
     * @return GameItem struct containing item details
     */
    function getItemDetails(uint256 tokenId) public view returns (GameItem memory) {
        require(gameItems[tokenId].exists, "Token does not exist");
        return gameItems[tokenId];
    }

    /**
     * @dev Get all items owned by a user
     * @param user Address of the user
     * @return Array of token IDs owned by the user
     */
    function getUserItems(address user) public view returns (uint256[] memory) {
        return userItems[user];
    }

    /**
     * @dev Get trade details
     * @param tradeId ID of the trade
     * @return Trade struct containing trade details
     */
    function getTradeDetails(uint256 tradeId) public view returns (Trade memory) {
        return trades[tradeId];
    }

    /**
     * @dev Get the total number of items created
     */
    function getTotalItems() public view returns (uint256) {
        return _tokenIdCounter;
    }

    /**
     * @dev Get the total number of trades created
     */
    function getTotalTrades() public view returns (uint256) {
        return _tradeIdCounter;
    }

    /**
     * @dev Get owner of a token
     */
    function ownerOf(uint256 tokenId) public view returns (address) {
        require(gameItems[tokenId].exists, "Token does not exist");
        return gameItems[tokenId].owner;
    }

    /**
     * @dev Cancel an active trade (only seller can cancel)
     * @param tradeId ID of the trade to cancel
     */
    function cancelTrade(uint256 tradeId) public {
        Trade storage trade = trades[tradeId];
        
        require(trade.seller == msg.sender, "Only seller can cancel");
        require(trade.status == TradeStatus.Open, "Trade is not open");

        trade.status = TradeStatus.Cancelled;

        // Return item ownership to seller
        gameItems[trade.tokenId].owner = trade.seller;
        userItems[trade.seller].push(trade.tokenId);
    }

    /**
     * @dev Internal function to remove item from user's item list
     */
    function _removeFromUserItems(address user, uint256 tokenId) internal {
        uint256[] storage items = userItems[user];
        for (uint256 i = 0; i < items.length; i++) {
            if (items[i] == tokenId) {
                items[i] = items[items.length - 1];
                items.pop();
                break;
            }
        }
    }

    /**
     * @dev Withdraw accumulated platform fees (only owner)
     */
    function withdrawFees() public onlyOwner {
        payable(owner).transfer(address(this).balance);
    }

    /**
     * @dev Get contract balance
     */
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
<img width="1920" height="1080" alt="Screenshot 2025-09-27 135838" src="https://github.com/user-attachments/assets/e151171c-b62d-43ea-80ce-c99354978cf1" />
