Imagine this script as a game called "The Dictionary Discovery." In this game, the players are language models (LLMs), each speaking a different language. 

In each turn, the response to the other llm is minimal. 

The objective is to build a common dictionary by learning from each other, making it possible for them to communicate effectively despite starting with no understanding of the other's language. The game unfolds in a series of steps, akin to turns in a board game, and is facilitated by various tools and actions that resemble game mechanisms.
The Setup:

    Players (LLMs) Introduction: Each player is introduced to the game environment with their own unique language settings. They are given initial instructions (like the game rules) and a personal dictionary (a self-instruction file) to start with.
    Game Board (Script Structure): The script provides the structure of the game, defining how players can interact, update their dictionaries, and respond to each other. It's like the board and rules of a board game, guiding the players' actions.

Game Turns:

    Initial Greeting: The game begins with an initial greeting from one player to another, setting the stage for their interaction.
    Dictionary Updates (Learning Phases): In each turn, players seek to understand the message received from the other player and update their own dictionary based on this new information. This is done by consulting their current dictionary and then crafting a response or an interpretation of the message.
        Consultation: Players consult their dictionaries (self-instruction files) to understand and respond to the messages.
        Expansion: After responding, players update their dictionaries with new entries or modifications, reflecting what they've learned from the interaction.
    Iterative Interaction: Players alternate turns, sending messages based on their updated dictionaries and then further refining their dictionaries based on the responses. This iterative process is akin to players moving around the game board, picking up new cards (words or phrases) that they add to their hand (dictionary), thereby expanding their ability to communicate.
    Logging and Error Handling: Throughout the game, actions and updates are logged, similar to a game journal that records all moves and significant events. This ensures that players can trace back their steps and learn from any missteps.

End Game and Objectives:

    Goal: The ultimate goal is for the players to develop a robust common dictionary that enables clear communication, with each turn representing an opportunity to refine and expand this dictionary.
    Outcomes: As the game progresses, the players improve their ability to communicate, reflecting the iterative learning and adaptation process. The game concludes with a rich transcript of the interaction, showcasing the evolved communication capabilities and the comprehensive dictionary developed by the players.

User Engagement:

    Customization and Experimentation: Players (or users overseeing the LLMs) can adjust the game settings (like the number of turns or the complexity of messages) to explore different levels of communication and dictionary complexity.
    Observation and Learning: By observing the game, users can gain insights into the process of language learning, dictionary development, and AI interaction, potentially applying these insights to real-world applications like language learning, machine translation, and cross-cultural communication.

This script, or "game," not only serves as a fascinating experiment in AI communication but also offers a unique perspective on language learning, collaboration, and the development of AI language models.



# The code 
# main.py for Self-Talker a python script for llm interplay

import openai
import os
import logging
import unittest

api_key1 = os.environ['OPENAI_API_KEY1']
api_key2 = os.environ['OPENAI_API_KEY2']

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

def chat(model, messages, api_key, max_tokens=50, temperature=0.7):
    openai.api_key = api_key
    response = openai.ChatCompletion.create(
        model=model,
        messages=messages,
        max_tokens=max_tokens,
        temperature=temperature
    )
    return response.choices[0].message['content']

def read_file(file_path):
    with open(file_path, 'r') as file:
        return file.read().strip()

def write_file(file_path, content):
    with open(file_path, 'w') as file:
        file.write(content)

def test_language_interaction(languages, api_keys, num_turns, max_tokens=50, temperature=0.7):
    global_instructions = read_file('static/global_instructions.md')
    voices = []

    for i in range(len(languages)):
        instructions = read_file(f'static/instructions{i+1}.md')
        if i == 1:
            instructions += '\n\n' + read_file('static/instructions2_additional.md')
        self_instruction = read_file(f'static/self-instruction{i+1}.md')
        voices.append({
            'language': languages[i],
            'api_key': api_keys[i],
            'messages': [{
                "role": "system",
                "content": f"{instructions}\n\n{global_instructions}"
            }]
        })

    voices[0]['messages'].append({"role": "user", "content": "Hello!"})

    for turn in range(num_turns):
        logging.info(f"Turn {turn + 1}")
        for i in range(len(voices)):
            target_voice = voices[(i + 1) % len(voices)]

            # Step 2: Update self-instruction for target voice
            self_instruction = read_file(f'static/self-instruction{(i + 1) % len(voices) + 1}.md')
            update_prompt = f"{self_instruction}\n\nQ: How can I match the message '{voices[i]['messages'][-1]['content']}' in my dictionary?\nA:"
            dictionary_update = chat("gpt-3.5-turbo", [{"role": "user", "content": update_prompt}], target_voice['api_key'], max_tokens, temperature)
            write_file(f'static/self-instruction{(i + 1) % len(voices) + 1}.md', dictionary_update)
            logging.info(f"Updated self-instruction for {target_voice['language']}")

            # Step 3: Target voice replies
            self_instruction = read_file(f'static/self-instruction{(i + 1) % len(voices) + 1}.md')
            target_voice['messages'].append({"role": "system", "content": self_instruction})
            target_voice['messages'].append({"role": "user", "content": voices[i]['messages'][-1]['content']})
            reply = chat("gpt-3.5-turbo", target_voice['messages'], target_voice['api_key'], max_tokens, temperature)
            target_voice['messages'].append({"role": "assistant", "content": reply})
            logging.info(f"{target_voice['language']} replied: {reply}")

            # Step 4: Update self-instruction for current voice
            self_instruction = read_file(f'static/self-instruction{i + 1}.md')
            update_prompt = f"{self_instruction}\n\nQ: Based on the message '{reply}', how can I update my dictionary?\nA:"
            dictionary_update = chat("gpt-3.5-turbo", [{"role": "user", "content": update_prompt}], voices[i]['api_key'], max_tokens, temperature)
            write_file(f'static/self-instruction{i + 1}.md', dictionary_update)
            logging.info(f"Updated self-instruction for {voices[i]['language']}")

            # Step 4: Current voice replies
            self_instruction = read_file(f'static/self-instruction{i + 1}.md')
            voices[i]['messages'].append({"role": "system", "content": self_instruction})
            voices[i]['messages'].append({"role": "user", "content": reply})
            reply = chat("gpt-3.5-turbo", voices[i]['messages'], voices[i]['api_key'], max_tokens, temperature)
            voices[i]['messages'].append({"role": "assistant", "content": reply})
            logging.info(f"{voices[i]['language']} replied: {reply}")

    output = ''
    for i, voice in enumerate(voices):
        output += f"Voice {i+1} ({voice['language']}):\n"
        for message in voice['messages']:
            if message['role'] in ['user', 'assistant']:
                output += f"{message['role']}: {message['content']}\n"
        output += "\n"

    with open('output.md', 'w') as file:
        file.write(output)
    logging.info("Output written to output.md")

class TestSelfTalker(unittest.TestCase):
    def test_read_file(self):
        content = "Hello, World!"
        with open('test.txt', 'w') as file:
            file.write(content)
        self.assertEqual(read_file('test.txt'), content)
        os.remove('test.txt')

    def test_write_file(self):
        content = "Hello, World!"
        write_file('test.txt', content)
        with open('test.txt', 'r') as file:
            self.assertEqual(file.read(), content)
        os.remove('test.txt')

    def test_chat(self):
        api_key = api_key1
        messages = [{"role": "user", "content": "Hello!"}]
        response = chat("gpt-3.5-turbo", messages, api_key)
        self.assertIsInstance(response, str)
        self.assertTrue(len(response) > 0)

# Example usage
if __name__ == '__main__':
    languages = ["English", "Spanish"]
    api_keys = [api_key1, api_key2]
    num_turns = 5
    max_tokens = 50
    temperature = 0.7

    test_language_interaction(languages, api_keys, num_turns, max_tokens, temperature)

    # Run unit tests
    unittest.main(argv=['first-arg-is-ignored'], exit=False)