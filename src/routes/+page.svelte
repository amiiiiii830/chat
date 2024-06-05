<script lang="js">
    import { onMount } from "svelte";

    const prompt1 = `
    The re-entrancy attack is a common attack against Solidity smart contracts. It allows the attacker to steal money from a smart contract by calling a vulnerable function in the contract repeatedly.
 Comment

Let's say that contract A calls contract B. Reentracy exploit allows B to call back into A before A finishes execution.

In the next example, EtherStore is a contract where you can deposit and withdraw ETH. This contract is vulnerable to re-entrancy attack.

contract EtherStore {
    mapping(address => uint256) public balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() public {
        uint256 bal = balances[msg.sender];
        require(bal > 0);

        (bool sent,) = msg.sender.call{value: bal}("");
        require(sent, "Failed to send Ether");

        balances[msg.sender] = 0;
    }

    // Helper function to check the balance of this contract
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
The Attack smart contract will be able to withdraw more than it deposited into EtherStore.

contract Attack {
    EtherStore public etherStore;
    uint256 public constant AMOUNT = 1 ether;

    constructor(address _etherStoreAddress) {
        etherStore = EtherStore(_etherStoreAddress);
    }

    // Fallback is called when EtherStore sends Ether to this contract.
    fallback() external payable {
        if (address(etherStore).balance >= AMOUNT) {
            etherStore.withdraw();
        }
    }

    function attack() external payable {
        require(msg.value >= AMOUNT);
        etherStore.deposit{value: AMOUNT}();
        etherStore.withdraw();
    }

    // Helper function to check the balance of this contract
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
Let's see how.

Deploy EtherStore
Deposit 1 Ether each from Account 1 (Alice) and Account 2 (Bob) into EtherStore
Deploy the Attack contract with address of EtherStore
Call Attack.attack sending 1 ether (using Account 3 (Eve)).
You will get 3 Ethers back (2 Ether stolen from Alice and Bob,
plus 1 Ether sent from this contract).
The reason is that Attack was able to call EtherStore.withdraw multiple times before
EtherStore.withdraw finished executing.

Here is how the functions were called

Attack.attack
EtherStore.deposit
EtherStore.withdraw
Attack fallback (receives 1 Ether)
EtherStore.withdraw
Attack.fallback (receives 1 Ether)
EtherStore.withdraw
Attack fallback (receives 1 Ether)
To prevent re-entrancy attacks, you need to

Ensure all state changes happen before calling external contracts
Use function modifiers that prevent re-entrancy
Below is an example showing how you might use the nonReentrant modifier on the EtherStore.withdraw function to prevent re-entrancy attacks. The attacker will not be able to execute the function concurrently.

contract EtherStore {

    bool internal locked;

    modifier nonReentrant() {
        require(!locked, "Reentrant call detected!");
        locked = true;
        _;
        locked = false;
    }
    
    function withdraw() public nonReentrant {
        // Implemenetation
    }    
}
Given the above context, please evaluate the Solidity code snippets and files from the user input to check for re-entrancy vulnerbilities. If you found a re-entracy vulnerbility, please also suggest how to fix it using the function modifier.
    `;

    const prompt2 = `
    In Solidity, smart contracts can be deleted from the blockchain by calling selfdestruct. However, the selfdestruct forcefully sends all remaining Ether stored in the contract to a designated address. It sends the Ethers even if the the contract at the designated address cannot handle the incoming Ether. That could mess up internal states of the designated contract, causing problems.
 Comment

For example, the following EtherGame contract allows every player to deposit 1 Ether. The 7th player will be the winner and can widthdraw 7 Ethers.

contract EtherGame {
    uint256 public targetAmount = 7 ether;
    address public winner;

    function deposit() public payable {
        require(msg.value == 1 ether, "You can only send 1 Ether");

        uint256 balance = address(this).balance;
        require(balance <= targetAmount, "Game is over");

        if (balance == targetAmount) {
            winner = msg.sender;
        }
    }

    function claimReward() public {
        require(msg.sender == winner, "Not winner");

        (bool sent,) = msg.sender.call{value: address(this).balance}("");
        require(sent, "Failed to send Ether");
    }
}
The Attack contract selfdestruct and forcefully sends Ether to the EthereGame contract so that it can never meet the 7 Ether requirement to declare a winner.

contract Attack {
    EtherGame etherGame;

    constructor(EtherGame _etherGame) {
        etherGame = EtherGame(_etherGame);
    }

    function attack() public payable {
        // You can simply break the game by sending ether so that
        // the game balance >= 7 ether

        // cast address to payable
        address payable addr = payable(address(etherGame));
        selfdestruct(addr);
    }
}
The attack works like the following.

Deploy EtherGame
Players (say Alice and Bob) decides to play, deposits 1 Ether each.
Deploy Attack with address of EtherGame
Call Attack.attack to forcefully sending 5 ether. This will break the game. No one can become the winner.
The solution is for EtherGame to NOT to rely on address(this).balance for its application logic as it can be forcefully changed by Attack. Here is a better version of EtherGame.

contract EtherGame {
    uint256 public targetAmount = 3 ether;
    uint256 public balance;
    address public winner;

    function deposit() public payable {
        require(msg.value == 1 ether, "You can only send 1 Ether");

        balance += msg.value;
        require(balance <= targetAmount, "Game is over");

        if (balance == targetAmount) {
            winner = msg.sender;
        }
    }

    function claimReward() public {
        require(msg.sender == winner, "Not winner");

        (bool sent,) = msg.sender.call{value: balance}("");
        require(sent, "Failed to send Ether");
    }
}
Given the above context, please evaluate the Solidity code snippets and files from the user input to check for selfdestruct vulnerbilities. If you found a vulnerbility, please also suggest how to fix it by avoiding relying on address(this).balance as discussed above.
    `;

    const prompt3 = `
    While a private field in a Solidity smart contract is not accessible from another smart contract, attackers can still easily figure out the value in the private field by examining the raw blockchain transactions stored in the public nodes. As a rule, you should be very careful with the private field, and avoid storing any sensitive information in smart contracts.
 Comment

Given the above context, please evaluate the Solidity code snippets and files from the user input to check for private fields in smart contarcts. If you find any, please ask the user to double check and make sure no confidential information is stored in the private fields.    
    `;

    const prompt4 = `
    The Solidity delegatecall is tricky to use and wrong usage or incorrect understanding can lead to devastating results.
 Comment

You must keep 2 things in mind when using delegatecall

delegatecall preserves context (storage, caller, etc…)
storage layout must be the same for the contract calling delegatecall and the contract getting called
Here is an example. HackMe is a contract that uses delegatecall to execute code. It is not obvious that the owner of HackMe can be changed since there is no function inside HackMe to do so.

contract Lib {
    address public owner;

    function pwn() public {
        owner = msg.sender;
    }
}

contract HackMe {
    address public owner;
    Lib public lib;

    constructor(Lib _lib) {
        owner = msg.sender;
        lib = Lib(_lib);
    }

    fallback() external payable {
        address(lib).delegatecall(msg.data);
    }
}
However an attacker can hijack the contract by exploiting delegatecall.

contract Attack {
    address public hackMe;

    constructor(address _hackMe) {
        hackMe = _hackMe;
    }

    function attack() public {
        hackMe.call(abi.encodeWithSignature("pwn()"));
    }
}
Here is how the attack works.

Alice deploys Lib
Alice deploys HackMe with address of Lib
Eve deploys Attack with address of HackMe
Eve calls \`Attack.attack()\`\`
Attack is now the owner of HackMe
Eve called Attack.attack().
Attack called the fallback function of HackMe sending the function selector of pwn(). HackMe forwards the call to Lib using delegatecall.
Here msg.data contains the function selector of pwn().
This tells Solidity to call the function pwn() inside Lib.
The function pwn() updates the owner to msg.sender.
Delegatecall runs the code of Lib using the context of HackMe.
Therefore HackMe's storage was updated to msg.sender where msg.sender is the
caller of HackMe, in this case Attack.

To prevent these attacks, the library contract called from delegatecall should be stateless and does not store any data field.

Given the above context, please evaluate the Solidity code snippets and files from the user input to check for delegatecall vunerabilities on stateful library contracts. If you find any, please suggest the user to refactor the library contract to stateless.
    `;

    let messages = [];

    function setTemplate(template) {
        if (template !== undefined) {
            document.getElementById("prompt").value = template;
        } else {
            console.error("Template is undefined");
        }
    }
    async function sendPrompt(event) {
        event.preventDefault();

        const promptElement = document.getElementById("prompt");

        if (!promptElement.value.trim()) return;

        const prompt = promptElement.value;

        const body = {
            model: "Codestral-22B-v0.1-hf-Q5_K_M",
            messages: [{ role: "user", content: prompt }],
        };

        const response = await fetch(
            "https://0x57fa64fb75d1b8c778063adcd81d99e525b6197d.gaianet.network/v1/chat/completions",
            {
                method: "POST",
                headers: {
                    "Content-Type": "application/json",
                    Authorization: `fake_token`,
                },
                body: JSON.stringify(body),
            },
        );

        if (response.ok) {
            const data = await response.json();
            const messageContent = data.choices[0].message.content;

            messages = [...messages, { prompt, response: messageContent }];
            promptElement.value = "";
        } else {
            console.error("Failed to fetch data:", response.statusText);
        }
    }

    function saveMessages() {
        let textToSave = "";
        messages.forEach(({ prompt, response }) => {
            textToSave += `Prompt:\n${prompt}\n\nResponse:\n${response}\n\n---\n\n`;
        });

        const blob = new Blob([textToSave], { type: "text/plain" });
        const link = document.createElement("a");
        link.download = "prompts_and_responses.txt";
        link.href = window.URL.createObjectURL(blob);
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
    }

    onMount(async () => {
        document
            .getElementById("promptForm")
            .addEventListener("submit", sendPrompt);
    });
</script>

<h3>Click to use any of the examples:</h3>

<title>Chat Template</title>

<div class="template-buttons">
    <div class="template-button" on:click={() => setTemplate(prompt1)}>
        Re-Entrancy Attack Detection<br /><small
            >Evaluate Solidity code for re-entrancy vulnerabilities and suggest
            fixes using function modifiers</small
        >
    </div>
    <div class="template-button" on:click={() => setTemplate(prompt2)}>
        Selfdestruct Vulnerability Check<br /><small
            >Identify selfdestruct vulnerabilities in Solidity contracts and
            recommend avoiding reliance on balance</small
        >
    </div>
    <div class="template-button" on:click={() => setTemplate(prompt3)}>
        Private Field Sensitivity Audit<br /><small
            >Review Solidity contracts for private fields to ensure no sensitive
            information is stored</small
        >
    </div>
    <div class="template-button" on:click={() => setTemplate(prompt4)}>
        Delegatecall Security Analysis<br /><small
            >Assess delegatecall usage in Solidity contracts to prevent stateful
            library exploitation by refactoring to stateless libraries</small
        >
    </div>
</div>

<form id="promptForm">
    <h3>Prompt:</h3>
    <label for="prompt">
        <textarea name="prompt" rows="10" id="prompt"></textarea>
    </label>
    <button type="submit">Send</button>
</form>

{#each messages as { prompt, response }}
    <div class="message">
        <pre><strong>Prompt:</strong> {prompt}</pre>
        <pre><strong>Response:</strong> {response}</pre>
    </div>
{/each}
<button on:click={saveMessages}> Save Prompts </button>

<style>
    textarea {
        width: 80%;
        min-height: 20em;
    }
    .message {
        width: 80%;
        margin-top: 20px;
        max-height: fit-content;
    }
    .template-buttons {
        display: grid;
        grid-template-columns: repeat(2, 1fr); /* Two columns */
        gap: 20px; /* Gap between buttons */
        margin-bottom: 20px;
    }
    .template-button {
        background-color: #f5d7b4; /* Adjust color as needed */
        border-radius: 0.25rem;
        padding: 0.5rem;
        cursor: pointer;
    }
</style>
