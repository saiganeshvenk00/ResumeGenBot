# ResumeGenBot
An AI-powered Telegram bot that tailors your resume to any job post in minutes. Just send a job description link and the bot scrapes it, extracts keywords, rewrites your resume bullets using GPT, and delivers a customized, ATS-friendly PDF directly to your chat.

**Step 1:**  
Trigger the workflow when a user sends a message on Telegram. The message (typically a job link) is forwarded through the flow.

**n8n Node:** Telegram Trigger  
- Watches for new incoming messages from a connected Telegram bot.


**Step 2:**  
Scrape the job description content from the URL received via Telegram. The scraped content is returned in markdown format and will be used in downstream AI steps.

**n8n Node:** HTTP Request (`Firecrawl Scraping`)  
- Method: `POST`  
- URL: `https://api.firecrawl.dev/v1/search`  
- Authentication: Header Auth (Firecrawl API key)

**Request Body:**
- <place holder>

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

**Step 4:**  
Convert the raw output from the keyword extraction agent into a structured JSON object. This parsed output can be referenced cleanly in future steps for folder naming, resume customization, and more.

**n8n Node:** Code (`First Keys`)  
- Type: JavaScript Code node  
- Function: Parses a plain-text string of `key: value` pairs into a valid JSON object.

**Code: <Placeholder>**

**Step 5:**  
Create a new folder in Google Drive named after the job title or company name. This keeps each tailored resume and its related files organized by application.

**n8n Node:** Google Drive (`Job Folder`)  
- Operation: Create Folder  
- Folder Name: Dynamically pulled from a parsed field like `{{ $json["Role name"] }}` or `{{ $json["Company name"] }}`  
- Parent Folder: Your predefined root directory for job folders in Google Drive

**Step 6:**  
Duplicate your resume template into the job-specific folder created in the previous step. The template contains placeholders (like `{exp1bullet1}`) that will be updated in later steps.

**n8n Node:** Google Drive (`Duplicate Resume Template`)  
- Operation: Copy File  
- Source File: Your original Google Docs resume template with placeholder tags  
- Destination Folder: The newly created folder from Step 5


**Step 7:**  
Use OpenAI to optimize your resume bullets using keywords extracted from the job description. The agent also references your original resume to maintain tone and structure. It rewrites existing bullets, adds new ones where placeholders are left blank, and recommends five role-specific tools.

**n8n Node: OpenAI Tools Agent (`Resume Builder`)** 
- This agent takes the extracted keywords and your reference resume as inputs and outputs an updated list of bullet points and tools.
<placeholder>
- To enable dynamic bullet injection, the resume template used in this workflow should contain placeholders (e.g., {dellbullet1}, {custexpbullet2}, {supertutor1}) instead of fixed bullet content. These placeholders should replace each resume bullet in the original document, while maintaining section structure, job titles, and formatting. Leave some placeholders intentionally blank (e.g., {dellbullet5}) to allow the AI to generate new, role-aligned bullets. The placeholder names must exactly match the keys returned by the AI agent to ensure accurate mapping when the document is updated later in the workflow.

**User Message Prompt:**
I am a job seeker who is applying for jobs in the market right now. My top roles are, Product Management, Product Marketing Management, AI Solutions Architect, Solutions Architecture and Solutions Engineering. You are an expert resume writer who is tasked at making my resume align with the job description without taking away the essence of my experience.

Your task is to take the input of key terms from a scraped job description ({{ $('Firecrawl Scraping').item.json.data }}) and my resume bullets that I will share below. You will need to include the skills (that are part of the job description) listed out in {{ $('First Keys').item.json['Top 10 technical skills needed'] }}. You will also need to include the tools listed in {{ $('First Keys').item.json['Top 10 technical tools needed'] }}. You will use the OpenAI model attached to optimize and give me an output consisting of optimized bullet points that are in line with the keywords pulled from the JD. Here are the conditions for each bullet:

Each bullet must be less than 125 characters (including spaces and punctuations) in length. Ensure any bullet generated has the same length as the others

Start with an extremely strong action verb

Must be quantized and must help the reader understand what I accomplished and the value I brought to the role

Follow the STAR format while phrasing the bullet. Must be situation, task, accomplishment, and result

The objective is to ensure the ATS system picks the resume because it has all the keywords. Basically, it must all fit in the same line and must not go to a second line. Every bullet must be the same length. Don’t alter the bullets too much, just a few words here and there to ensure my original resume is still intact. But ensure the resume aligns perfectly with the keywords provided. The resume bullets must be only as long as the bullets I am sharing with you. I am sharing them in the following format: bullet name : bullet content.

You can find the resume bullets in the "Reference Resume Doc" tool attached. The document includes 9 sections representing prior experiences, each with a set of bullet points that describe accomplishments. A few placeholder bullets ({exp1bullet5} and {exp3bullet3}) are intentionally left blank to give you room to generate additional content aligned to the role.

The sections and bullet placeholders are structured as follows:

Technical Solutions Consultant ({exp1bullet1} through {exp1bullet5})
Solutions Engineering Intern ({exp2bullet1}, {exp2bullet2})
Support Specialist | Infrastructure ({exp3bullet1} through {exp3bullet3})
Automation Platform Startup ({exp4bullet1} through {exp4bullet3})
EdTech Volunteer Product Manager ({exp5bullet1}, {exp5bullet2})
AI Tool Builder | Workflow Automation ({exp6bullet1}, {exp6bullet2})
Client Onboarding Platform ({exp7bullet1}, {exp7bullet2})
Resume Assistant Project ({exp8bullet1}, {exp8bullet2})
Robotics Strategy Capstone ({exp9bullet1}, {exp9bullet2})

Give me all the optimized bullets in the following format. No intro, no line breaks, just:
bullet_name : new bullet

Also, suggest 5 tools I will need for the Role:{{ $('Firecrawl Scraping').item.json.data[0].title }} based on {{ $('First Keys').item.json['Top 10 technical tools needed'] }}. Output format:
tool1 : Name
tool2 : Name
tool3 : Name
tool4 : Name
tool5 : Name

Don’t include '\n' at the end of bullets.

**Step 8:**  
Parse the optimized bullet output from OpenAI into a structured JSON object. The output from the AI agent is a flat string of key-value pairs (e.g., `dellbullet1: updated bullet`). This code step converts that into a format suitable for updating the Google Doc.

**n8n Node:** Code (`Sort Bullets`)  
- Type: JavaScript Code node  
- Function: Parses AI output line by line and converts it to JSON.

**Code:<placeholder>**


---

**Step 9:**  
Update the duplicated resume template with the optimized bullet points. This node maps the JSON keys (e.g., `dellbullet1`, `custexpbullet3`) to the placeholders in the Google Doc and replaces them with the AI-generated content.

**n8n Node: Google Docs (`New Resume`)**
- Operation: Update Document  
- Input: JSON object from Step 8  
- Action: Replaces all matching placeholders inside the document with new bullet content

> To enable dynamic bullet injection, the resume template used in this workflow should contain placeholders (e.g., `{dellbullet1}`, `{custexpbullet2}`, `{supertutor1}`) instead of fixed bullet content. These placeholders should replace each resume bullet in the original document, while maintaining section structure, job titles, and formatting. Leave some placeholders intentionally blank (e.g., `{dellbullet5}`) to allow the AI to generate new, role-aligned bullets. The placeholder names must exactly match the keys returned by the AI agent to ensure accurate mapping when the document is updated later in the workflow.

**Step 10:**  
Convert the updated Google Doc resume into a downloadable PDF file. This creates a polished, ready-to-send resume version aligned with the job description.

**n8n Node:** Google Drive (`Download as PDF`)  
- Operation: Download File  
- File: The updated Google Doc from Step 9  
- Format: PDF

**Step 11:**  
Upload the finalized PDF resume back into Google Drive for record-keeping or future reference. This step ensures each job-specific resume is stored in its respective job folder.

**n8n Node:** Google Drive (`Upload as PDF`)  
- Operation: Upload File  
- File: The PDF downloaded in Step 10  
- Destination: The same job-specific folder created in Step 5

**Step 12:**  
Send the final, tailored PDF resume back to the user via Telegram. This allows for instant access and reuse without needing to leave the chat.

**n8n Node:** Telegram (`Send Message`)  
- Operation: Send Document  
- Input: PDF file from Step 10  
- Output: Sends the resume directly into the same chat where the job link was submitted
