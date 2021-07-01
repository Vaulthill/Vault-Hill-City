//SPDX-License-Identifier: Unlicensed
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/utils/Counters.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";

/*
implemetation of VHC token
*/
contract VaultHillCity is Ownable, ERC1155 {
    
    using Counters for Counters.Counter;
    using SafeMath for uint256;
    Counters.Counter private tokenId;
    
    uint256 public FungibleTokenId;
    uint256 maxId = 10002;
    address owner_;
    
    uint256 public FTTotalSupply = 340000000 *10 **18;
    uint256 public FTInitialSupply = 17000000 *10 **18;
    uint256 public FTCirculatingSupply = FTInitialSupply;
    
    event eventOwnerRoyalty(uint256 tokenId, uint256 royaltyAmount);
    event eventCreatorRoyalty(uint256 tokenId, uint256 royaltyAmount);
    
    constructor (string memory _uri) ERC1155("") {
        owner_ = _msgSender();
        tokenId.increment();
        FungibleTokenId = tokenId.current();
        ipfsHash[FungibleTokenId] = _uri;
        _mint(msg.sender, FungibleTokenId, FTInitialSupply, "");
        ownersToken[msg.sender].push(FungibleTokenId);
    }
    
     mapping (uint256 => string) ipfsHash;
     mapping(address => uint[]) public ownersToken;
     mapping (uint256 => uint256) creatorRoyalty;
     mapping (uint256 => uint256) ownerRoyalty;
     /**
     * @dev Creates `amount` tokens of token type `id`, and assigns them to `account`.
     * @param _uri A unique identifier of what the token "looks" like. A URI could be an API call over HTTPS, an IPFS hash, or anything else unique.
     * @param _account The address where ownership of NFT will be Transferred. e.g: creator
     * @param _amount The amount of tokens to be generated for each id. In case of NFT, generate only 1 token.
     * @param _data Additional data with no specified format.
     * @return True if the operation was successful.
     */ 
        function mintNFT(string memory _uri, address _account, uint256 _amount, bytes memory _data) public virtual returns(uint256) {
            tokenId.increment();
            uint256 newTokenId = tokenId.current();
            require(newTokenId < maxId, "Max cap reached: No more NFTs can be minted");
            ipfsHash[newTokenId] = _uri;
            _mint( _account,  newTokenId,  _amount,  _data);
            ownersToken[_account].push(newTokenId);
            return newTokenId;
        }
        /**
         * this function will mint Fungible tokens. e.g: VHC tokens
         **/
        
        function mintFT(uint256 _amount) public onlyOwner returns(uint256) {
            require(FTCirculatingSupply.add(_amount *10 **18) <= FTTotalSupply, "Max cap reached");
            FTCirculatingSupply = FTCirculatingSupply.add(_amount *10 **18);
            _mint(msg.sender, FungibleTokenId, _amount *10 **18, "");
            return FTCirculatingSupply;
        }
        /**
         * @return Array of NFT ids of each address
         */
        function tokensOfOwner(address _owner)public view returns(uint[]memory){
            return ownersToken[_owner];

        }
        /**
         * @return Uri data of each id.
         */
         function uri(uint  _id)public override view returns (string memory){
            return ipfsHash[_id];
        }
        
        /**
         * more suitable method
          **/
        function setRoyaltyForEachNFT(uint256 _id, uint256 _ownerShare, uint256 _creatorShare) public {
            ownerRoyalty[_id] = _ownerShare *10 **18;
            creatorRoyalty[_id] = _creatorShare *10 **18;
            emit eventOwnerRoyalty(_id, _ownerShare);
            emit eventCreatorRoyalty(_id, _creatorShare);
        }
        
        function getOwnerRoyalty(uint _id) public view returns(uint256) {
            return ownerRoyalty[_id];
        }
        
        function getCreatorRoyalty(uint256 _id) public view returns(uint256) {
            return creatorRoyalty[_id];
        }
        
        /**
         * This function will transfer NFT from one account to another account, distributing creator and owner royalty too.
         **/
        function transferNFT(address _creator, address _from, address _to, uint _id, uint256 _amount, bytes memory _data) public returns(bool) {
            require(_from != address(0), "Sender address cannot be zero");
            require(_to != address(0), "Recipient address cannot be zero");
            require(_amount > 0, "Transfer amount must be greater than zero");
            
            uint256 ownerShare = ownerRoyalty[_id];
            uint256 creatorShare = creatorRoyalty[_id];
            
            if(_id != FungibleTokenId) {
                require(balanceOf(_from, _id) == 0, "NFT not found.");
                require(balanceOf(_from, FungibleTokenId) >= ownerShare.add(creatorShare), "Insufficient VHC balance");
                
                safeTransferFrom(_from, _creator, FungibleTokenId, creatorShare, _data);
                safeTransferFrom(_from, owner_, FungibleTokenId, ownerShare, _data);
                safeTransferFrom(_from, _to, _id, _amount, _data);
            }
            return true;
        }      
        /**
         * This function will batch transfer NFT from one account to another account, distributing creator and owner royalty too.
         **/
        function transferBatchNFT(address _creator, address _from, address _to, uint256[] memory _ids, uint256[] memory _amounts, bytes memory _data) public {
            require(_from != address(0), "caller cannot be zero address");
            require(_to != address(0), "Receiver cannot be zero");
            require(_ids.length == _amounts.length, "ids quantity must be equal to amount quantity");
            
            uint256 totalOwnerShare;
            uint256 totalCreatorShare;
            for(uint256 i = 0; i < _ids.length; i++) {
                uint256 ownerShare = ownerRoyalty[_ids[i]];
                uint256 creatorShare = creatorRoyalty[_ids[i]];
                
                totalOwnerShare = totalOwnerShare.add(ownerShare);
                totalCreatorShare = totalCreatorShare.add(creatorShare);   
            }
            
            safeTransferFrom(_from, owner_, FungibleTokenId, totalOwnerShare, _data);
            safeTransferFrom(_from, _creator, FungibleTokenId, totalCreatorShare, _data);
            safeBatchTransferFrom(_from, _to, _ids, _amounts, _data);
        }
        
        
        
        
        
        
        
        
        
        
        
}