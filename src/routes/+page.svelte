<script lang="js">
    import { onMount } from "svelte";

    let messages = [];
    let prompt1 = "";
    let prompt2 = "";
    let prompt3 = "";
    let prompt4 = "";
    function setTemplate(template) {
        document.getElementById("prompt").value = template;
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
        const [prompt1, prompt2, prompt3, prompt4] = await Promise.all([
            fetch("/src/assets/prompt1.txt").then((res) =>
                res.ok ? res.text() : Promise.reject("prompt1 not found"),
            ),
            fetch("/src/assets/prompt2.txt").then((res) =>
                res.ok ? res.text() : Promise.reject("prompt2 not found"),
            ),
            fetch("/src/assets/prompt3.txt").then((res) =>
                res.ok ? res.text() : Promise.reject("prompt3 not found"),
            ),
            fetch("/src/assets/prompt4.txt").then((res) =>
                res.ok ? res.text() : Promise.reject("prompt4 not found"),
            ),
        ]);
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
        <textarea name="prompt" rows="4" id="prompt"></textarea>
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
        width: 60%;
    }
    .message {
        width: 60%;
        margin-top: 20px;
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
