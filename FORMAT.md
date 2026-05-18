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

# Mapping

todo: write mapping documentation

# mszdlg1

todo: write msdlg1 documentation (low priority)