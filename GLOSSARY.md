# Shopping List

A personal shopping-list application where people keep private lists of things to buy and, over time, share them with others.

## Language

### Lists and items

**List**:
A named collection of items with its own access control. Private to its owner until shared.
_Avoid_: Checklist, shopping list (when meaning one specific list)

**Item**:
Something that needs to be bought, kept on a list. An item persists after it is purchased and can be needed again.
_Avoid_: Product, entry, line

**Purchased**:
The state of an item that has been bought. Purchased items stay on the list in a separate section and can be returned to unpurchased.
_Avoid_: Bought, checked, done, ticked

### People and access

**Owner**:
The person who created a list. Manages who has access to it and can delete it.
_Avoid_: Admin, creator

**Editor**:
A person with access to a list who may change its items.
_Avoid_: Collaborator, contributor, member

**Invite link**:
A link that grants access to a list when redeemed by a signed-in person.
_Avoid_: Share link, invitation code

### Conversation and history

**Comment**:
A message attached to a list or an item. A question is a comment, and a reply is a comment with a parent comment.
_Avoid_: Note, message, question

**Activity**:
The recorded history of changes made to a list.
_Avoid_: History, log, audit trail
