# Notion Voice Tasks

Create new tasks in Notion using your voice.

## What Is This?

Notion Voice Tasks is a simple Express app that allows you to create new tasks in Notion through voice assistants like Siri and Google Assistant.

It uses ChatGPT to parse natural language, extracting the following information for each task:

- Task Name
- Assignee
- Due Date
- Project

You can create multiple tasks through a single voice command. After tasks are completed, you'll get a confirmation message reporting on the number of tasks created, the AI cost, and the elapsed time.

Since tasks typically do not require a lot of words to describe, this workflow usually costs less than $0.001 (1/10th of a cent) to run.

## How Does It Work?

Under the hood, this app accepts a POST request sent to its `/task` endpoint. The body must be a JSON object containing four keys: `task` (string), `name` (string), `date` (ISO 8601 string), and `secret`. Example:

```
{
  task: 'I need to clean my room tomorrow and Tony needs to mount the new hair light for studio design by next wednesday and Mariiss needs to book flight by june 2 and Ransom needs to finish writing the learning hacks script draft by june 10',
  name: 'Thomas Frank',
  date: '2023-05-25T18:59:56-06:00',
  secret: 'someuniquephrase'
}
```

The `secret` key must match the `NOTION_VOICE_TASKS_SECRET` variable in your `.env` file; otherwise, the request will be rejected. This prevents others from sending requests even if they have the webhook URL.

The `name` property is used to set an assignee when words like "I need to" are used (e.g. you can assign a task to yourself). The `date` property is used to allow for relative due-date assignments (e.g. "I need to do X next Friday). ChatGPT will only know what "next Friday" means if we also include the current date, along with the timezone.

When a payload is received, the following happens:

1. Each key's value in the payload is validated and sanitized.
2. The sanitized JSON object is combined with the current date/timezone to form a prompt. This is sent to as a Chat request to ChatGPT, along with a customized system message that instructs ChatGPT to act as a natural-language task parser.
3. ChatGPT returns a JSON array containing an object for each task.
4. The cost of the request and response are calculated (typically this is < $0.001).
5. The ChatGPT response is validated, primarily to make sure the response is valid JSON. If it isn't, steps are taken to attempt JSON repair.
6. The app queries Notion to fetch a list of all workspace users and all projects in the Projects database. We then use fuzzy search (Fuse.js) to determine the closest match for both the **assignee** and the **project**. If none of the fetched data meets a search-score threshold, neither of these values will be set. This ensures that tasks go to your Inbox by default, rather than being assigned to the wrong person or project. *You can adjust the threshold in `src/config/config.js`.* **Note:** This is also why the app sets the **Source** property's value. In the case that a task is incorrectly assigned, you'll always be able to find it by creating a view of your Tasks database that shows all tasks with a **Source** property that matches the one you set for your workflow (by default, it's "iOS Shortcut").
7. For each task, an object is constructed that meets the format required by the Notion API.
8. Each task is sent to Notion, creating a new page in the Tasks database.
9. A confirmation message is sent back to the source of the POST request (e.g. your phone).

## How Can I Use This?

The easiest way to use this workflow is to use the Pipedream automation I built that replicates its functionality. To get started, check out my full tutorial for setting that up. (Links coming soon)

I built this Express app mostly as a learning exercise, as I wanted to "graduate" from building competely linear workflows in Pipedream to learning how to build a fully functional, modular app.

I have not yet deployed this app to a host; I've only done local testing using ngrok.

If you want to use this, you'll need to do the following:

**First, proceed with caution. I am sharing this project to help others learn. This should be considered to be one dude's learning project, shared to the internet. It is NOT production-ready software. I make NO guarantees about its functionality or safety. I've taken steps to sanitize/validate input, but I am no computer security expert. Make sure you know what you're doing if you want to run it locally, as doing so involves exposing your computer to the internet. ABSOLUTELY NO SUPPORT WILL BE PROVIDED FOR THIS CODE.**

With that out of the way, download/fork the project so you have the code on your local machine. You'll need node.js installed, along with npm.

Install the dependencies:

```
npm install
```

Create a `.env` file in the project's root directory with the following keys:

```
NOTION_TASKS_DB = 
NOTION_PROJECTS_DB = 
NOTION_API_KEY = 
OPEN_AI_KEY = 
MAX_TOKENS = 2000
NOTION_VOICE_TASKS_SECRET =
```

Create a new internal integration at your [integrations page](https://www.notion.so/my-integrations). Learn how to do this [here](https://thomasjfrank.com/notion-api-crash-course/#create-a-notion-integration).

Give it permission to read user data, and set the integration key as your `NOTION_API_KEY`.

Give the integration permission to access your **tasks** and **projects** databases, or ideally the page that contains both of them. Learn how to do this [here](https://thomasjfrank.com/notion-api-crash-course/#add-your-integration-to-your-pokedex-database).

Get the databases IDs for your task/project databases and add them to your `.env` file. [Learn how to find your database IDs here](https://thomasjfrank.com/notion-api-crash-course/#obtain-your-database-id).

Your Notion tasks database must have the following properties (names must match exactly):

| Property Name | Type                     |
|---------------|--------------------------|
| Name          | Title (default property) |
| Assignee      | Person                   |
| Due           | Date                     |
| Project       | Relation                 |
| Source        | Select                   |

As shown in the table above, make sure your Tasks database has a **Select** property named **Source**. The `src/services/formatChatResponse.js` file sets a value for this property on each created page (task). The value is defined in `src/config/config.js`.

You will also need an OpenAI account. While you may be given $5 in free credits upon signup, you'll need to add your billing details to continue using this beyond that. 

You'll need to obtain an [OpenAI API Key](https://platform.openai.com/account/api-keys) and set it in the `.env` file. [Learn how to obtain a key here](https://thomasjfrank.com/how-to-transcribe-audio-to-text-with-chatgpt-and-notion/#transcribe-the-audio-file-with-whisper).

To run an active web server with a webhook URL you can target with your Siri/Google Assistant automation, you can use [ngrok](https://ngrok.com/). You'll need an ngrok account to continue.

On your local machine, you can globally install ngrok:

```
npm install -g ngrok
```

Then you can connect your account:

```
ngrok authtoken your_auth_token
```

Start your web server. By default, this project creates one on port 3000. You can start it by running `npm run dev` in the Terminal (or use `nodemon` if you want the server to restart when you save code changes).

Once your web server is running, open a new Terminal instance and start ngrok:

```
ngrok http 3000
```

Once done, you'll see a URL like `https://xxxx-xx-xxx-xxxx-x-x.ngrok-free.app` that points to your localhost URL. Copy that URL; you'll use it as the destination for the POST request that you'll need to set up on Shortcuts (iOS/Siri) or Tasker (Android/Google Assistant). Note that you'll need to append `/task` to the end of that URL.

Finally, you'll need to set up a way to send valid POST requests to the app. At present, I have instructions on how to do this with **Shortcuts** (iOS, MacOS). My team and I have also successfully tested this with Tasker, enabling the creation of tasks via dictation on Android with Google Assistant. I'll release instructions on that soon.

More broadly, you can use *any* tool that is able to send a valid JSON body like the one shown above, containing `task`, `name`, and `date`, properties.

My real intention for this tool (and the associated Pipedream workflow) is not to build a "Siri to Notion" workflow, but rather to create a versatile pipeline that you can use to quickly create tasks via nearly any convenient tool – Siri, Google Assistant, Slack commands, a Raycast extension... the possibilities are numerous.

Here's how you'd build a Siri Shortcut to use with this workflow:

![Siri Shortcut configuration](https://github.com/TomFrankly/notion-voice-tasks/blob/main/img/Shortcut_Example.jpg)

## Learn More/Support My Work

You can learn more about Notion, and Notion automation, at these pages on my site:

* [Notion Fundamentals – Full Beginners' Series for Learning Notion](https://thomasjfrank.com/fundamentals/)
* [Notion Automations – Collection of No-Code and Code Tutorials](https://thomasjfrank.com/notion-automations/)
* [Notion API Guide – Full (and free) course for learning the Notion API](https://thomasjfrank.com/notion-api-crash-course/)

If you'd like to support my work, consider buying one of my Notion templates (or picking up one of the free ones):

* [Ultimate Brain – complete second brain for Notion](https://thomasjfrank.com/brain/)
* [Creator's Companion – the ultimate planning system for content creators](https://thomasjfrank.com/creators-companion/)
* [Free Notion Templates](https://thomasjfrank.com/templates/)

## License

[GNU v3.0](https://github.com/TomFrankly/notion-voice-tasks/blob/main/LICENSE)











