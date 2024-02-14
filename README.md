# Collab

An experimental collaborative document-editing over nostr.

## How it works

A user creates a new document, `kind:31111` and p-tags the editors the user wants to invite to collaborate.

The user shares the naddr of the event with the other collaborators.

When authors or collaborators are writing, ephemeral events are published constantly so that each one can see in kinda real-time the changes the others are making.

When collaborators want to propose a change they hit the "Save" button, which generates an event of the same kind as the original (31111 by default) *signed with their own key*.

The owner of the document can then choose to make the newest version the canonical version by resigning the event with the new content.
