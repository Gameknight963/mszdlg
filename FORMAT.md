# mszdlg2

### Core structure

A mszdlg file is a zip archive, so you can extract it to inspect it and edit it manually.

It includes two types of files: metadata (nodes.json) and audio files.

Please note that sometimes I don't follow c# naming conventions here, it is to follow JSON convention and to match the names I found in the decompiled code from Miside Zero.

### nodes.json

nodes.json is serialized from the DialoguePack class:

```csharp
public class DialoguePack
{
    public int PackFormat = 2;
    public string TargetGameVersion = Application.version;
    public List<DialogueTreeDTO> trees;
}
```

`PackFormat = 2` is for mszdlg2. `TargetGameVersion` should be "Alpha 0.72," but it may differ. Currently all mszdlg2 packs should work fine on all versions (0.7-0.72), although using a pack on a version other than it's `TargetGameVersion` will give you a warning.

#### DialogueTreeDTO

A Data Transfer Object (DTO) representing the DialogueTree class. Serializes from this class:

```csharp
public class DialogueTreeDTO
{
    public List<DialogueNodeDTO> nodes;
    public List<int> startNodeIds;
    public float? chirpTime;
    public float? initialDelay;
    public float? exitDelay;
    public string name;
}
```

Of note: mszdlg2 and 1 replace the direct node references with an ID to eliminate recursion issues. This id is generated at mapping time by [MSZDialogueMapper](https://github.com/Gameknight963/MSZDialogueMapper). More on the mapping protocol [here](#Mapping).

`nodes` is all the nodes the tree contains.

`startNodeIds` represents the IDs of the nodes that will start this tree.

`chirpTime` is, well, chirp time. If it's null, Miside Zero Dialogue Override won't patch it. Same for any other nullable fields.

`initialDelay` is the delay when entering the tree.

`exitDelay` is the delay when exiting the tree.

`name` is a purely cosmetic property and does not affect gameplay. Used to make UI more readable in the editor.

#### DialogueTreeDTO

A DTO representing a dialogue node. Serializes from this class:

```csharp
public class DialogueNodeDTO
{
    public int id;
    public int[] nextNodeIds;
    public string dialogueText;
    public string speakerName;
    public float delay;
}
```

Again, you will notice that each node now has an ID, and instead of direct references in NextNodes, it uses an array of IDs.

Miside Zero can afford to put next nodes as `DialogueNode[]` since `DialogueNode` is a reference type and there would be no recursion. However, when we serialize to JSON, we don't have the luxury of reference types, so I needed another solution that would avoid recursion and wasn't super wasteful.

`id` is the node's ID.

`nextNodeIds` is the IDs of the next possible nodes branching from this node.

`dialogueText` is the text that will be displayed when this node is played.

`speakerName` is who speaks the node, used by Miside Zero to get chirp sounds.

`delay` is how long the game will wait after the node is played before beginning the next node.

#### Example nodes.json

```json
{
  "PackFormat": 2,
  "TargetGameVersion": "Alpha 0.72",
  "trees": [
    {
      "name": "Start",
      "nodes": [
        {
          "id": 0,
          "nextNodeIds": [],
          "dialogueText": "Looks like I made it alright.",
          "speakerName": "Kiri",
          "delay": 0.5
        }
      ],
      "startNodeIds": [0]
    }
  ]
}
```

### Audio files

Audio files are stored alongside nodes.json, like this:

```csharp
$"{treeIndex}_{nodeIndex}"
```

Examples: 

- `0_12.wav`

- `0_4.mp3`

Miside Zero Dialogue Override uses bass to import audio, so it supports anything bass supports (a lot.)

# Dialogue Export System

If you want to make your own mapper, it is important you follow this specification, otherwise things will break. So here's now it works step by step:

## 1. Tree discovery

All dialogue trees must be collected using:

```csharp
GameObject.FindObjectsOfType<DialogueTree>()
```

This call defines the complete export set. Any tree not returned here will not be included in the `DialoguePack`. The order of returned objects also influences tree ordering inside the final pack, since they are matched back up again by their index in the array returned by this call, so it is important that you get every tree to avoid screwing up the index of other trees.

Make sure you call this when scene "Version 1.9 POST" is active otherwise you won't get any nodes since obviously the other scenes don't have dialogue.

## 2. Node flattening

Each `DialogueTree` is converted into a flat list of nodes:

```csharp
List<DialogueNode> treeNodes = tree.GetAllNodes();
```

This list is the **single source of truth** for all subsequent mapping.

The ordering of `treeNodes` must be:

- stable
- deterministic
- identical between export runs

If ordering changes, all node IDs and links will change, ruining everything.

#### How this ordering is produced

The flattening process works like this:

```csharp
foreach (DialogueNode firstNode in tree.startNodes)
    TraverseNode(firstNode, visited);
```

Each start node becomes a traversal root. From there, nodes are explored recursively:

```csharp
if (node.nextNodes != null)
    foreach (DialogueNode next in node.nextNodes)
        TraverseNode(next, visited);
```

`TraverseNode` is a recursive depth-first graph walk with deduplication. It builds the final node list by expanding connections outward from a starting node while ensuring each node is only recorded once.

Here’s the function:

```csharp
public static List<DialogueNode> TraverseNode(DialogueNode node, List<DialogueNode> visited)
{    
    if (node == null || visited.Contains(node)) return visited;

    visited.Add(node);

    if (node.nextNodes != null)
        foreach (DialogueNode next in node.nextNodes)
            TraverseNode(next, visited);

    return visited;
}
```

Importantly, although it may seem like an obvious choice, **do not use a hashset** for this function since the order matters.

## 3. Node ID assignment

Each node is assigned an ID based on its position in `treeNodes`.

Node IDs are local to the tree only and have no global meaning. This is a bit limited as it doesn't let you cross reference trees. 

mszdlg3 may have nodes be referenced with their tree index **and** their local index, making cross-referencing between trees possible. Depends if it becomes nessecary or not.

This currently does not cause any issues in game since all nodes point to nodes within their tree.

## 4. Connection mapping

Node connections are serialized by converting object references into indices:

```csharp
nextNodeIds = node.nextNodes?
    .Where(n => n != null)
    .Select(n => treeNodes.IndexOf(n))
    .ToArray();
```

Meaning `nextNodes` (object references) turn into `nextNodeIds` (integer indices).

All referenced nodes must exist inside `treeNodes`. Otherwise, the connection will not resolve correctly. Again, this may change in mszdlg3.

## 5. Start node mapping

Entry points are handled using the same indexing system:

```csharp
startNodeIds = tree.startNodes
    .Where(n => n != null)
    .Select(n => treeNodes.IndexOf(n))
    .ToList();
```

## 6. Tree packaging

Add each tree to `pack.trees` independently:

```csharp
pack.trees.Add(new DialogueTreeDTO { ... });
```

## Notes

If you didn't read anything else, read this.

- Trees must be discovered via `GameObject.FindObjectsOfType<DialogueTree>()` or something that returns the same values as it
- `GetAllNodes()` must return a consistent ordering
- Avoid hashsets since indexes matter a lot here. On my machine, MSZDialogueMapper can export in ~40ms using lists. So it's fine. 

Any deviation from these rules will result in broken graph reconstruction.

# mszdlg1

todo: write msdlg1 documentation (low priority)