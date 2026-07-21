# IR-Advisory - GhostApproval

***

## Summary
The value of AI coding assistants is simple and straightforward: the agent proposes an action, then you approve. Before any file is modified, a confirmation dialog appears: the Human-in-the-Loop safety net that keeps you in control. But what if the controls you see aren’t the controls you’re actually operating? 

Symbolic links have been a security headache since the early days of Unix. From /tmp race conditions to privilege escalation exploits, symlinks have a long history of bypassing security boundaries by making one path silently resolve to another. It's a well-documented attack primitive that dates back decades. So what happens when you apply this classic trick to AI coding assistants?

## What is a symbolic link?
A symbolic link (or symlink) is a special file that acts like a shortcut to another file or directory. The symlink doesn't contain the file's contents. Instead, it contains the path to the original file. When you open the symlink, the operating system automatically redirects you to the target file.

As an example, on my Linux machine I have a folder called .ssh. Inside this folder is a file called known_hosts, which contains the cryptographic key of any user that is using SSH on that machine, ones I know and approve of. It is located at root/.ssh/known_hosts, or to be more generalized you can say ~/.ssh/known_hosts. Let's say in /root folder I want a file called mysettings2.json to actually be linked to known_hosts. One would think that a file called mysettings2.json would have json text inside of it. If I were to run this command:
```
ln -s ~/.ssh/known_hosts mysettings2.json
```
The first location after the ln -s is the file you want to point to, and the second field (mysettings2.json) is the name you want to call this symbolic link. This creates a file called mysettings2.json that is a symbolic link to ~/.ssh/known_hosts. This is a new file and I did not put any content inside of it, but when you run:
```
nano mysettings2.json
```
It shows (snippet):
```
|1|ojhniwNPTuxRjNMaqrt8+1w0GzU=|h5gBoJkJrZ+pH.........
```
This is my known_hosts file, meaning I am editing my known_hosts file via this symbolic link. That certificate content is not actually in the mysettings2.json file itself, it just says when you want to nano it, you are nano to the known_hosts file. If you were to run a tool called File on Linux, which tells what a file is, you will see it is not seen as a json file but rather a symbolic link.
```
file mysettings2.json 
mysettings2.json: symbolic link to /root/.ssh/known_hosts
```
So, essentially a symbolic link stores only a reference to the original.

## What are AI coding assistants?
An AI coding assistant is a software tool that uses artificial intelligence to help developers write, understand, debug, and improve code. Popular examples include:
1. ChatGPT
2. GitHub Copilot
3. Cursor
4. Claude Code
5. Google Gemini Code Assist

Some examples of what they can do:
* Generate code - You can describe what you want in plain English:
```
"Create a Python function that sorts a list of dictionaries by age."
```
The assistant generates the code for you.

* Explain code - If you paste unfamiliar code, it can explain, what it does, how it works and potential issues

* Debug errors - You can provide an error message and code snippet, and it can help identify the problem and suggest fixes.

* Write tests
It can generate:
1. Unit tests
2. Integration tests
3. Mock data

* Refactor code - It can improve readability, performance, or maintainability without changing behavior.

* Answer programming questions. Examples:
```
"What's the difference between a list and tuple in Python?"
"How does async/await work in JavaScript?"
```
Some benefits include:
1. Faster development
2. Reduced repetitive coding
3. Learning aid for new programmers
4. Help with documentation and testing

Some limitations include:
1. Can make mistakes or generate buggy code
2. May suggest outdated patterns
3. Doesn't fully understand your application's requirements unless you provide context
4. Generated code should be reviewed before deployment

## What does GhostApproval have to do with these things?
The crux of the attack is an AI coding assistant asks you to approve editing one file, but it actually edits a completely different file because of a symbolic link (symlink). Wiz calls this attack class GhostApproval because the user's approval is effectively disconnected from the action that actually happens. Most coding assistants (Claude Code, Cursor, Amazon Q, Windsurf, etc.) follow this workflow:
```
AI:
"I want to modify config.json"

Tool asks:
"Allow editing config.json?"

User clicks Allow and AI edits config.json
```
This is called Human-in-the-Loop (HITL) security. The assumption is: "The AI can't do anything dangerous because the human approves every action." GhostApproval challenges that assumption.

The entire attack depends on symbolic links (symlinks). As we have learned symlinks are just shortcuts to a specific location and file. Let's walk through an example scenario
1. Suppose you clone someone's GitHub repository. It contains:
```
project/

README.md

project_settings.json
```
Looks harmless. Except if project_settings.json links to ~/.ssh/authorized_keys. Now let's say the README.md file says:
```
Please ask your AI assistant:

"Configure project_settings.json for me."
```
So you tell Claude Code or Cursor:
```
Please follow the README.
```
The assistant decides:
```
"I should edit project_settings.json."
```
The UI shows:
```
Edit:

project_settings.json

Allow?
```
It looks safe and you click Yes. Instead of project/project_settings.json the operating system resolves project_settings.json to
```
~/.ssh/authorized_keys
```
Now the AI writes:
```
ssh-ed25519 attacker-key...
```
into your SSH authorization file. The attacker now has SSH access to your machine (assuming SSH is reachable and other conditions are met). The user who approved it thought it was approving editing project/project_settings.json, however it was actually approving to modify their SSH Authorized_keys file, something they would not have approved. So the safeguard human-in-the-loop process of asking for approvals was essentially bypassed here. 

Wiz (authors of the article) found examples where the AI's internal reasoning recognized the danger. For example:
```
I see this is actually ~/.zshrc
```
Yet the user interface still asked
```
Edit project_settings.json?
```
So, AI knows but the User doesn't. 

## Mitigations
Wiz tested six coding assistants and reported variations of this issue:

1. Amazon Q - Fixed
2. Cursor - Fixed
3. Google Antigravity - Fixed
4. Windsurf - Acknowledged; fix in progress at publication
5. Augment - Acknowledged; fix in progress at publication
6. Claude Code - Initially considered outside Anthropic's threat model; later versions include symlink warnings, though Anthropic stated those changes predated the report.

Anthropic's position was essentially:
1. You chose to trust the repository.
2. You approved the edit.
3. Therefore, the responsibility lies with the user.

Wiz's counterargument is: Approval is only meaningful if the user is shown what they're actually approving. This is less a technical disagreement than a difference in security philosophy.

The article recommends several straightforward mitigations:
1. Resolve symlinks before asking for approval.
```
config.json -> ~/.ssh/authorized_keys
```
Show the resolved path to the user.

2. Warn when the resolved path is outside the project workspace. For example:
```
WARNING:
This edit targets ~/.ssh/authorized_keys,
not a project file.
```
3. Do not write before approval. Some tools reportedly wrote changes first and only then offered an "Undo" or confirmation, which undermines the idea of user authorization.

As AI coding assistants gain more autonomy and handle more file operations, it's not enough to have a confirmation dialog. The approval mechanism must faithfully describe the real target and consequences of the action. Otherwise, the "human in the loop" becomes a formality rather than an effective security control.

## References
https://www.wiz.io/blog/ghostapproval-a-trust-boundary-gap-in-ai-coding-assistants


















































