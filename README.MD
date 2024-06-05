# Chat to Test Your Prompts

This is a minimal tool for testing prompts with the newly released Codestral-22B model. You can test your prompt, observe its result, and improve it iteratively.

The Codestral-22B model is hosted on the Gaianet AI network. This tool is hard-coded to work exclusively with this backend. It is intended for testing purposes among open LLM enthusiasts.

## How to Use

1. Install dependencies:
    ```sh
    npm install
    ```

2. Start the development server:
    ```sh
    npm run dev
    ```

3. Open your browser and navigate to `http://localhost:5173`.

4. Click on the "Save Prompts" button to save the conversation history as a text file (`prompts_and_responses.txt`).