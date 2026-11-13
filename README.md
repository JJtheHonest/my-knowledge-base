# My Knowledge Base

A personal knowledge base built with **Basic Memory**, **Obsidian**, and **Markdown**, with **iCloud Drive** for access across Apple devices and **GitHub** for version control and backup.

This repository brings together study notes, technical explanations, and hands-on practice in AI, agent engineering, reinforcement learning, programming, and software engineering.

## How It Is Built

The setup uses a single collection of Markdown files shared by several tools:

| Component | Role |
| --- | --- |
| Markdown | Stores the notes as portable, readable text files. |
| Obsidian | Provides the interface for reading, editing, organizing, and linking notes. |
| Basic Memory | Indexes the notes and connects them to AI assistants through the Model Context Protocol (MCP). |
| iCloud Drive | Synchronizes the Obsidian vault across Apple devices for access and editing. |
| Git and GitHub | Track changes and maintain a versioned backup in a private repository. |

### Local Notes and Obsidian

The knowledge base lives in a local folder opened as an Obsidian vault. Notes are organized into topic directories and connected through internal links. Markdown keeps the content accessible outside Obsidian as well.

### Basic Memory and AI Assistants

Basic Memory is configured to use the same note directory. It indexes the Markdown content so connected AI assistants can search existing notes, retrieve relevant context, and help create or update notes through MCP.

This makes the collection useful both for manual study in Obsidian and as persistent context for AI-assisted learning and technical work.

### iCloud Across Apple Devices

The Obsidian vault is stored in iCloud Drive and synchronized across my Apple devices. This allows me to read and edit the same collection of notes from multiple devices using Obsidian.

iCloud handles file synchronization, while Git records explicit snapshots of the knowledge base.

### GitHub Backup

The vault is also a Git repository connected to a private GitHub repository. Commits preserve the history of changes, and pushes back up those committed snapshots to GitHub.

Device-specific Obsidian workspace state, caches, and macOS metadata are excluded through ignore rules.

## Everyday Workflow

1. Read, write, and connect notes in Obsidian on an Apple device.
2. Let iCloud synchronize the files across devices.
3. Use an AI assistant connected to Basic Memory to find relevant context and help develop or update notes.
4. Review changes, commit them with Git, and push to GitHub.

## Topics

- Artificial intelligence and large language models
- Agent engineering and multi-agent systems
- Reinforcement learning and agentic RL
- Programming languages and development tools
- Software engineering and system design

The collection grows with ongoing study and project work. Notes capture both conceptual understanding and practical experience.
