# BaseLegends
BaseLegends
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract BaseLegends is ERC721, Ownable, ReentrancyGuard {

    uint256 public constant MAX_SUPPLY = 10000;
    uint256 public constant TIER_SIZE = 1000;
    uint256 public constant PRICE_PER_TIER = 0.00024 ether;

    uint256 public totalMinted;
    string public baseURI;

    event Minted(address indexed minter, uint256 tokenId, uint256 price);

    constructor() ERC721("BaseLegends", "BLEG") Ownable(msg.sender) {
        baseURI = "https://api.baselegends.xyz/metadata/";
    }

    function mint(uint256 quantity) external payable nonReentrant {
        require(totalMinted + quantity <= MAX_SUPPLY, "Exceeds max supply");
        require(quantity > 0 && quantity <= 10, "Max 10 per transaction");

        uint256 totalPrice = _calculatePrice(quantity);
        require(msg.value >= totalPrice, "Insufficient ETH");

        for (uint256 i = 0; i < quantity; i++) {
            uint256 tokenId = totalMinted;
            _safeMint(msg.sender, tokenId);
            totalMinted++;

            emit Minted(msg.sender, tokenId, _getCurrentPrice());
        }

        // Refund excess ETH
        if (msg.value > totalPrice) {
            payable(msg.sender).transfer(msg.value - totalPrice);
        }
    }

    function _calculatePrice(uint256 quantity) internal view returns (uint256) {
        uint256 totalPrice = 0;
        for (uint256 i = 0; i < quantity; i++) {
            uint256 currentTier = (totalMinted + i) / TIER_SIZE;
            totalPrice += currentTier * PRICE_PER_TIER;
        }
        return totalPrice;
    }

    function _getCurrentPrice() internal view returns (uint256) {
        uint256 currentTier = totalMinted / TIER_SIZE;
        return currentTier * PRICE_PER_TIER;
    }

    function getCurrentMintPrice() external view returns (uint256) {
        return _getCurrentPrice();
    }

    function getTierInfo() external view returns (
        uint256 currentTier,
        uint256 currentPrice,
        uint256 mintedInTier
    ) {
        currentTier = totalMinted / TIER_SIZE;
        currentPrice = currentTier * PRICE_PER_TIER;
        mintedInTier = totalMinted % TIER_SIZE;
        return (currentTier, currentPrice, mintedInTier);
    }

    // Owner functions
    function ownerMint(uint256 quantity) external onlyOwner {
        require(totalMinted + quantity <= MAX_SUPPLY, "Exceeds max supply");
        
        for (uint256 i = 0; i < quantity; i++) {
            _safeMint(msg.sender, totalMinted);
            totalMinted++;
        }
    }

    function setBaseURI(string calldata newBaseURI) external onlyOwner {
        baseURI = newBaseURI;
    }

    function withdraw() external onlyOwner {
        payable(owner()).transfer(address(this).balance);
    }

    function _baseURI() internal view override returns (string memory) {
        return baseURI;
    }

    receive() external payable {}
}
