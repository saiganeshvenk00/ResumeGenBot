# ResumeGenBot
---

![ResumeGenBot](https://github.com/user-attachments/assets/f5e7aaa3-14fb-4f80-bd46-24dabd9dc58d)

**An AI-powered Telegram bot that tailors your resume to any job post in minutes. Just send a job description link and the bot scrapes it, extracts keywords, rewrites your resume bullets using GPT, and delivers a customized, ATS-friendly PDF directly to your chat.**

**Step 1:**  

Trigger the workflow when a user sends a message on Telegram. The message (typically a job link) is forwarded through the flow.

**n8n Node:** Telegram Trigger  
- Watches for new incoming messages from a connected Telegram bot.

---

**Step 2:**  

Scrape the job description content from the URL received via Telegram. The scraped content is returned in markdown format and will be used in downstream AI steps.

**n8n Node:** HTTP Request (`Firecrawl Scraping`)  
- Method: `POST`  
- URL: `https://api.firecrawl.dev/v1/search`  
- Authentication: Header Auth (Firecrawl API key)

**Request Body:**
- <place holder>

---

**Step 3:**  

Use OpenAI to extract structured keywords and metadata from the scraped job description. This includes key ATS-relevant elements such as role title, company name, team, skills, and tools.

**n8n Node:** OpenAI Tools Agent (`Keyword Generator`)  
- This agent processes the markdown job description and returns a flat list of key-value pairs, each on its own line.

**User Message Prompt:**
You are a helpful assistant. Based on the information in {{ $json.data[0].markdown }}, return a flat list of key-value pairs. The pairs must be key terms that an ATS resume system will immediately pick up on based on the scraped job description in {{ $json.data[0].markdown }}

Each item must be one line only in the format: Key: Value.
Do not include any line breaks (\n) or lists in the values. If there are multiple values, separate them by commas in the same line.

Required Keys:

Company name
Role name (if not available, predict what it can be)
Team name (if available, else write "Not specified")
Top 10 technical skills needed (according to JD and your suggestions)
Top 10 technical tools needed (be specific on the kind of tools an expert in this role would use. Be specific and name the tool, not what kind of tool it is)

---

**Step 4:**  

Convert the raw output from the keyword extraction agent into a structured JSON object. This parsed output can be referenced cleanly in future steps for folder naming, resume customization, and more.

**n8n Node:** Code (`First Keys`)  
- Type: JavaScript Code node. Find code [here](http://github.com/saiganeshvenk00/ResumeGenBot/blob/main/FirstKeysNode)
- Function: Parses a plain-text string of `key: value` pairs into a valid JSON object.



---

**Step 5:**  

Create a new folder in Google Drive named after the job title or company name. This keeps each tailored resume and its related files organized by application.

**n8n Node:** Google Drive (`Job Folder`)  
- Operation: Create Folder  
- Folder Name: Dynamically pulled from a parsed field like `{{ $json["Role name"] }}` or `{{ $json["Company name"] }}`  
- Parent Folder: Your predefined root directory for job folders in Google Drive

---

**Step 6:**  

Duplicate your resume template into the job-specific folder created in the previous step. The template contains placeholders (like `{exp1bullet1}`) that will be updated in later steps.

**n8n Node:** Google Drive (`Duplicate Resume Template`)  
- Operation: Copy File  
- Source File: Your original Google Docs resume template with placeholder tags  
- Destination Folder: The newly created folder from Step 5

---

**Step 7:**  

Use OpenAI to optimize your resume bullets using keywords extracted from the job description. The agent also references your original resume to maintain tone and structure. It rewrites existing bullets, adds new ones where placeholders are left blank, and recommends five role-specific tools.

**n8n Node: OpenAI Tools Agent (`Resume Builder`)** 
- This agent takes the extracted keywords and your reference resume as inputs and outputs an updated list of bullet points and tools.

- To enable dynamic bullet injection, the resume template used in this workflow should contain placeholders (e.g., {dellbullet1}, {custexpbullet2}, {supertutor1}) instead of fixed bullet content. These placeholders should replace each resume bullet in the original document, while maintaining section structure, job titles, and formatting. Leave some placeholders intentionally blank (e.g., {dellbullet5}) to allow the AI to generate new, role-aligned bullets. The placeholder names must exactly match the keys returned by the AI agent to ensure accurate mapping when the document is updated later in the workflow. The following image is an example of how mine looks like:

![resume placeholder example](https://github.com/user-attachments/assets/f9902afa-14c6-4689-8656-82e54b5b5e16)

- User Message Prompt: Find prompt [here](https://github.com/saiganeshvenk00/ResumeGenBot/blob/main/ResumeBuilderPrompt)


---

**Step 8:**  
Parse the optimized bullet output from OpenAI into a structured JSON object. The output from the AI agent is a flat string of key-value pairs (e.g., `dellbullet1: updated bullet`). This code step converts that into a format suitable for updating the Google Doc.

**n8n Node:** Code (`Sort Bullets`)  
- Type: JavaScript Code node
- Type: JavaScript Code node. Find code [here]((https://github.com/saiganeshvenk00/ResumeGenBot/blob/main/SortBulletsNode)) 
- Function: Parses AI output line by line and converts it to JSON.


---

**Step 9:**  
Update the duplicated resume template with the optimized bullet points. This node maps the JSON keys (e.g., `dellbullet1`, `custexpbullet3`) to the placeholders in the Google Doc and replaces them with the AI-generated content.

**n8n Node: Google Docs (`New Resume`)**
- Operation: Update Document  
- Input: JSON object from Step 8  
- Action: Replaces all matching placeholders inside the document with new bullet content

> To enable dynamic bullet injection, the resume template used in this workflow should contain placeholders (e.g., `{dellbullet1}`, `{custexpbullet2}`, `{supertutor1}`) instead of fixed bullet content. These placeholders should replace each resume bullet in the original document, while maintaining section structure, job titles, and formatting. Leave some placeholders intentionally blank (e.g., `{dellbullet5}`) to allow the AI to generate new, role-aligned bullets. The placeholder names must exactly match the keys returned by the AI agent to ensure accurate mapping when the document is updated later in the workflow.

---

**Step 10:**  

Convert the updated Google Doc resume into a downloadable PDF file. This creates a polished, ready-to-send resume version aligned with the job description.

**n8n Node:** Google Drive (`Download as PDF`)  
- Operation: Download File  
- File: The updated Google Doc from Step 9  
- Format: PDF

---
**Step 11:**  


Upload the finalized PDF resume back into Google Drive for record-keeping or future reference. This step ensures each job-specific resume is stored in its respective job folder.

**n8n Node:** Google Drive (`Upload as PDF`)  
- Operation: Upload File  
- File: The PDF downloaded in Step 10  
- Destination: The same job-specific folder created in Step 5

---

**Step 12:**  
Send the final, tailored PDF resume back to the user via Telegram. This allows for instant access and reuse without needing to leave the chat.

**n8n Node:** Telegram (`Send Message`)  
- Operation: Send Document  
- Input: PDF file from Step 10  
- Output: Sends the resume directly into the same chat where the job link was submitted
