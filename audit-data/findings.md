### [H-1] Storing the password on-chain makes it visible to anyone, no longer private.

**Description:** All data stored on-chain is visible to anyone, and can be directly read from the blockchain. The `PasswordStore::s_password` variable is intended to be a private variable and only accessed via the `PasswordStore::getPassword` function, which is intended to be only called by the owner.

**Impact:** Anyone can read the private password, severly breaking the functionality of the protocol.

**Proof of Concept:** 
The below test case shows how anyone can read the password directly from the blockchain.

1. Create a locally running chain.
```bash
make anvil
```

2. Deploy the contract to the chain.
```bash
make deploy
```

3. Run the storage tool
```bash
cast storage <ADDRESS_HERE> 1 --rpc-url http://127.0.0.1:8545
```
We gave `1`, because `PasswordStore::s_password` is the storage slot on the contract.

you will get output like this;
`0x6d7950617373776f726400000000000000000000000000000000000000000014`

You can parse that hex to a string wtih:
```bash
cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
```
and you will get the output:
`myPassword`.

**Recommended Mitigation:** Due to this, the overall architecture of the contract should be rethought. One could encrypt the password off-chain, and then store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the password. However, you'd also likely want to remove the view function as you wouldn't want the user to accidentally send a transaction with the password that decrypts you password.




### [H-2] `PasswordStore::setPassword` has no access control, anyone can change owners password.

**Description:** The `PasswordStore::setPassword` is set to be an external password. But this function should allow only the owner to set a new password.

**Impact:** Anyone can change the password of the owner, severly breaking the contract intended functionality.

**Proof of Concept:** Add the following to the `PasswordStore.t.sol`.
<details>
<summary>Code</summary>

```js
    function test_anyone_can_set_password(address randomAddress) public {
        vm.assume(randomAddress != owner);
        vm.prank(randomAddress);
        string memory expectedPassword = "myNewPassword";
        passwordStore.setPassword(expectedPassword);

        vm.prank(owner);
        string memory actualPassword = passwordStore.getPassword();
        assertEq(actualPassword, expectedPassword);
    }
```
</details>

**Recommended Mitigation:** Add an access control conditional to the `setPassword` function.
```javascript
if (msg.sender != s_owner) {
    revert PasswordStore__NotOwner();
} 
```


### [I-1] The `PasswordStore::getPassword` NatSpec indicates a parameter that does not exist, causing the natspec to be incorrect.

**Description:** The `PasswordStore::getPassword` function signature is `getPassword()` while the natspec says it should be `getPassword(string)`.
```js
    /*
     * @notice This allows only the owner to retrieve the password.
 >>  * @param newPassword The new password to set.  <<
     */
    function getPassword() external view returns (string memory) {
```

**Impact:** NatSpec is incorrect.

**Recommended Mitigation:** Remove the incorrect natspec line.

```diff
-  * @param newPassword The new password to set.
```