# Personal Assistant
A personal AI assistant who analyzes personal data and provides answers to questions based on the data. Following are the capabilities of the Assistant:

* Analyze any text file uploaded by the User and extract key personal details. Update the DB with the key details.
* Update any key categories or personal details in the DB as per user prompt.
* Delete any particular user information from the DB as per the user prompt.
* Answer any questions about the personal details contained in the DB.
* General chat with Groq Llama LLM.

## Brief Overview of Used Frameworks
* I have used here the open-sourced Llama model (8-billion parameter) as the base LLM for this implementation. Currently, [Groq Cloud](https://console.groq.com/playground) provides a free API for its usage on their cloud.
* I have used [CrewAI](https://www.crewai.com/) open sourced library for creating all AI-agents using their crew-based AI agent orchestration flow. 
*  CrewAI is a cutting-edge framework for orchestrating role-playing, autonomous AI agents. It is built on top on [LangChain](https://www.langchain.com/) framework.
* I have used [Chroma DB](https://python.langchain.com/v0.2/docs/integrations/vectorstores/chroma/) for building our Vector Database.
* However, the original ChromaDB Langchain framework lacks certain features, hence, I have copied their github implementation and made necessary changes in the chroma file ```chroma_aya.py``` for it to work.
    * Currently, the Langchain Chroma framework does not support returning the indices of the entry in the Database for Vector Similarity Searches and other search features. Refer to [this link](https://api.python.langchain.com/en/latest/vectorstores/langchain_community.vectorstores.chroma.Chroma.html). Hence, for this use case, I have tweaked their functionality to return the indexes and metadata as part of the simialrity searches.
    * Main changes have been made in the following functions:
        * `_results_to_docs_and_scores`
        * `similarity_search`
        * `similarity_search_by_vector`
        * `similarity_search_by_vector_with_relevance_scores`
        * `similarity_search_with_score`


## How to run the StreamLit UI

1. Load all the necessary libraries. They have been mentioned in the ```requirements.txt```.
Make sure that the versions of ```LangChain``` and ```PyDantic``` match.

2. When everything is installed, set up an account on [Groq Cloud](https://console.groq.com/playground). 

3. Create an API key for the Llama 8B parameter version from the Groq cloud account. Set up the API key on your local machine using ```export GROQ_API_KEY=<your-api-key-here>```.

4. Until the time of writing setting up and using Groq for Llama is free. This may change over time. In that case, use some other keys and make changes likewise to the model inside the file ```utils.py```.

5. `cd AI-PersonalAssistant` and run `streamlit run gui.py` for the GUI version. Run `python3 non_gui.py` for the the Non-Gui version.

## Architectural Diagram and Workflow
![Alt text](https://github.com/ayanavasarkar/personal_assistant/blob/main/ui_imgs/monochrome_diagram.jpeg)

![Alt text](https://github.com/ayanavasarkar/personal_assistant/blob/main/ui_imgs/colorful_diagram.jpeg)

### Explanation of the Workflow:
- First the user enters the Groq API to get started.
- Next the user enters a query. Based on the user query, the `PromptClassifier` AI agent classifies the prompt into one of - `save something in memory`, `deduce memory from unstructured text`, `update memory`, 
            `delete memory`, or `off_topic`.
    - If it is classified as `off_topic`, then a general response is given by the LLM chat from the `generic_response` function inside `utils.py`. \
    *Eg- "How are you doing?"*

    - If it is classified as `save something in memory`, then the user must upload a text file along with it. \
    *Eg - Save the data to Memory*
        - If the input file is provided then the `ExtractFromUploadedFile` Agent extracts the necessary details and uploads it into the vector DB. The default path for the DB is set as `/tmp/db`. You can change it.
        - If the input file is not given, then the user is prompted to upload one.

    - If the prompt is classified as `deduce memory from unstructured text`, then we compose a Chroma DB vector retriever who uses the Llama LLM embeddings to retrieve a response to the query based on the database information. \

    *Currently the AI Agents have been specifically prompted to answer based on context only present in the DB. If no information is there in the DB then the Agent will output that it does not know the answer to the query.*
    *Eg - What is the email id of the person?*

    - If the query is classified as `update memory`, we extract the exact text from the DB based on `similarity_search` between the prompt and the database entries. Then we call the `MemoryManagement` Agent to delete locate the necessary detail in the text, remove the old value and enter the new value from the user query. Then we write back to the DB with the updated value. \
    *Eg - update the email id to test@g.com*

    - If the query is classified as `delete memory`, then the `CategoryExtraction` agent is called to find the exact ddetail or category to delete from the user data. Based on the detail extracted, we find the most similar text in the Chroma DB and use the `DeleteMemory` agent to delete only the necessary part. \
    *Eg - delete the email id*


### Files in this Repo:
- `gui.py` - It contains the gui-based implementation of the entire system.
- `non-gui.py` - It contains the non-gui-based implementation of the system. Currently it has a terminal-based UI.
- `utils.py` - The utility functions for model loading, Chroma DB loading, storing the db to disk, splitting raw text into chunks based on the `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)` before embedding the chunks and storing them in DB.
- `ai_agents.py` - Contains all the AI agent crews required for the various jobs in this implementation, as discussed above.
- `agents.py` - Contains all the initialization of the AI agents with specific prompts for each of their jobs.
- `tasks.py` - Contains the initialization and detailed prompts of the tasks of each of the agents, including, the exact input data, their detailed tasks and the expected output from each of the crews.
- `tests/test_jobs.py` - Contains the unit tests for each of the crews, agents and tasks.

### Note: Limitations
1. The system does not have memory of previous conversations and user queries yet. `StreamlitChatMessageHistory` has a functionality which integrates with the `CrewAI` library's `ConversationBufferMemory` to keep track of the user queries. However, there seems to be some discrepancies causing the auto functionality to non properly function.

2. The system currently only accepts ```.txt``` files. Any other file will throw an Exception and crash the GUI.

3. While inputing ```.txt``` files, the system does not check if the content is already present in the DB. Hence, if duplicat entry is given, there willbe duplicates in the DB.

4. The agents have been specifically prompted to not answer anything beyond the data present in the DB. Hence, for questions requiring some amount of analysis, the system does not answer them.

## Future Prospocts:
Following are the features that would be focused on in the near future:

1. Integrate the DB checking when new data is uploaded, so that duplicates are not present in the DB. This can be done in a variety of ways:
    - When data is extracted by the `ExtractFromUploadedFile` Agent, do a check across each entry in the DB using similarity to see if the new data entered is already present in the DB. However, this slows down the process and also a good threshold for the similarity threshold must be experimented with. Further, the proper embedding structure must also be tested. \
    *Based on some initial set of experiments, we can use the `SentenceTransformer` for embedding the input and using a postgres to store the embeddings of each category already in the DB. Then do just a quick similarity comparison to figure out if that entry is already present in the DB.*

    - Maintain another vector DB where the chuck sizes are small and after every N-seconds, the DB is loaded and duplicates are removed. (Easier to build and maintain, but absolutely Non-scalable for large systems.)

2. Currently, I record the history of the chat conversation, but the previous chats have no influence on the current prompt. Induce some form of history where the system has a short-term, long-term and eventual memory. Useful in the following case:
*User Prompt 1- What is the EMail ID?; AI ANSWERS - test@g.com* \
*User Prompt 2 - Can you update it?; AI ANSWER - WHat is it?*

3. Currently, this system functions as a Personal Assistant based on the details of only one-person. Extention beyond one-person to something like this for a family or a group of people would be interesting.