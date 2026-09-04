## Preamble
This is a project to be submitted for a course to build AI agents. You job is to - 
1. Tutor me and ensure I thoroughly understand the project scope, 
2. what the output should be, what the project is about and what we are building
3. IMPORTANTLY every single code snippet that is written for the project or inside the project folder. this is not limited to the diff analysis but handholding me in every step of the way as I would have to present this in front of a panel. 


## Project scaffolding 
This file is a marker or a guide as to what has to go inside the projects folder. the projects folder is what will be submited so ensure that has only project related files. 

The below information could be a repetion of what is inside the Project/Readme.md however this is just to prep you for what is to be done inside. 

You are free to prepare a claude.md in this root folder for your analysis and preparation. most importantly any tracker or progress of the tutorial and build. Go in phases and pick the build, explanantio for each phase as you see fit. make sure you document these in a seperate md file for a decluttered experience. 

You must move forward only when a phase is completely built, tested and understood by me. otherwise it cannot be marked complete. i am panning to sprint through this exercise in 5 days. prepare a plan accordingly with each day phase and duration of each phase.

## Project Scenario
You’ve been hired as an AI Engineer at a gaming analytics company developing an assistant called UdaPlay. Executives, analysts, and gamers want to ask natural language questions like:

“Who developed FIFA 21?”
“When was God of War Ragnarok released?”
“What platform was Pokémon Red launched on?”
“What is Rockstar Games working on right now?”
Your agent should:

Attempt to answer the question from internal knowledge (about a pre-loaded list of companies and games)
If the information is not found or confidence is low, search the web
Parse and persist the information in long-term memory
Generate a clean, structured answer/report
Project Specifications
In this project, you will build an AI Research Agent called UdaPlay designed to answer questions about video games. The agent will be capable of:

Answering user questions about games, including:

Game titles and their details
Release dates and platforms
Game descriptions and genres
Publisher information
Using a two-tier information retrieval system:

Primary: RAG (Retrieval Augmented Generation) over a local dataset of games
Secondary: Web search using the Tavily API when internal knowledge is insufficient
Implementing a robust evaluation system:

Assessing the quality of retrieved information
Determining when to fall back to web search
Providing confidence levels in answers
Generating clear, well-structured responses that:

Cite information sources
Combine information from multiple sources when needed
Present information in a natural, readable format

## For this project you'll use the following libraries:

chromadb>=1.0.4

openai>=1.73.0

pydantic>=2.11.3

python-dotenv>=1.1.0

tavily-python>=0.5.4

### Part 1 - RAG Pipeline
Set up a ChromaDB vector database
Process and embed game data from JSON files
Implement semantic search functionality
Create a reusable vector store manager

### Part 2 - Agent Implementation
Build an agent with three core tools:
retrieve_game: Search the vector database
evaluate_retrieval: Assess answer quality
game_web_search: Fall back to web search
Implement a state machine for agent workflow
Create a reporting system for clear output

## Suggestions to Make Your Project Stand Out
Personalize the Dataset: Add extra games, companies, or platforms to the dataset and demonstrate richer queries.

Advanced Memory: Implement persistent long-term memory so the agent “learns” from web searches.
Structured Output: Return answers in both natural language and structured JSON for easy integration.

Visualization: Create a dashboard or visualization of the agent’s retrieval process or knowledge base.

Custom Tools: Add extra tools, such as sentiment analysis of game reviews or trending games detection.

Eval : build evals to measure the quality of the prompts and the tool calling

## Project Rubric

## RAG

| Criteria | Submission Requirements |
|----------|------------------------|
| Prepare and process a local dataset of video game information for use in a vector database and RAG pipeline | - The submission includes the notebook (`Udaplay_01_solution_project.ipynb`) that loads, processes, and formats the provided game JSON files.<br>- The processed data is added to a persistent vector database (e.g., ChromaDB) with appropriate embeddings.<br>- The notebook or script demonstrates that the vector database can be queried for semantic search. |

## Agent Development

| Criteria | Submission Requirements |
|----------|------------------------|
| Implement agent tools for internal retrieval, evaluation, and web search fallback. | The submission includes at least three tools:<br>- A tool to retrieve game information from the vector database.<br>- A tool to evaluate the quality of retrieved results.<br>- A tool to perform web search using an API (e.g., Tavily).<br><br>Each tool is implemented as a function/class and is integrated into the agent workflow.<br><br>The agent:<br>- first attempts to answer using internal knowledge,<br>- evaluates the result,<br>- and falls back to web search if needed. |
| Build a stateful agent that manages conversation and tool usage. | - The agent is implemented as a class or function that maintains conversation state.<br>- The agent can handle multiple queries in a session, remembering previous context.<br>- The agent's workflow is implemented as a state machine or similar abstraction.<br>- The agent produces clear, structured, and well-cited answers. |
| Demonstrate and report on the agent's performance with example queries. | - The submission includes the notebook (`Udaplay_02_solution_project.ipynb`) that runs the agent on at least three example queries (e.g., about game release dates, platforms, or publishers).<br>- The output for each query includes the agent's reasoning, tool usage, and final answer.<br>- The report includes at least the response with citation, if any. |