# Notes on Notes.app
For future reference and to aid anyone else who might want to extract data from the "Notes" app. I compiled this out of curiosity and a desire to backup my notes content. 

This document only covers the current format of notes, as synced with iCloud. The application also supports IMAP synced notes with reduced functionality. They are stored in a separate database (and on the IMAP server). You're on your own there, but it's pretty much just MIME and HTML.

## OSX Files

The notes are stored in `~/Library/Group Containers/group.com.apple.notes` in a sqlite database named `NoteStore.sqlite`. The database contains sync state from iCloud. It's a CoreData store, but I approached it from plain sqlite.

We're interested in the `ZICCLOUDSYNCINGOBJECT` table and the `ZICNOTEDATA` table. The former contains the sync state of each iCloud object - notes, attachments, and folders, and the latter contains the note data. Everything has a UUID identifier `ZIDENTIFIER`. The `ZICCLOUDSYNCINGOBJECT` table column `ZNOTEDATA` points to the `Z_PK` column in `ZICNOTEDATA`.
The meat of the data is in zlib compressed protobuf blobs. For documents, it's `ZDATA` in `ZNOTEDATA` and for tables/drawings, it's `ZMERGEABLEDATA` in `ZICCLOUDSYNCINGOBJECT`.

Apple uses a [CvRDT](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) for syncing (evidenced by various strings appearing in the data and analysis of what happens as you edit). I suspect the one used for documents, which they call "topotext" is not actually a CvRDT (they seem to order conflicts on a first to sync basis rather than vector clock) but I could be mistaken. It doesn't matter since everything goes through iCloud, which serializes the merges.

## Protobuf Data

**Document Wrapper**
Everything is wrapped with a versioned document object:

    message Document {
        repeated Version version = 2;
    }
    message Version {
        optional bytes data = 3;
    }

There are additional fields that aren't relevant to us, and I've never seen more than one `version`. The content of the `data` field is also protobuf but varies depending on whether we're looking at a note, table, or drawing.

**Notes**
The protobuf data for a note is a `String` as described below. Assume everything is optional and I’ve elided the CRDT stuff.  (Repeated field 3 of String is a sequence clock, length, attribute clock, tombstone, and children.  It forms a DAG. For chunks of length >1 the clock is implicitly incremented for each character.)


    message String {
        string string = 2;
        // these are in order, disjoint, and their length sums to the length of string
        repeated AttributeRun attributeRun = 5;
    }
    message AttributeRun {
        uint32 length = 1;
        ParagraphStyle paragraphStyle = 2;
        Font font = 3;    
        uint32 fontHints = 5; // 1:bold, 2:italic, 3:bold italic
        uint32 underline = 6;
        uint32 strikethrough = 7;
        int32 superscript = 8; // sign indicates super/subscript
        string link = 9;
        Color color = 10;
        AttachmentInfo attachmentInfo = 12;
    }
    message ParagraphStyle {
        // 0:title, 1:heading, 4:monospace, 100:dotitem, 101:dashitem, 102:numitem, 
        // 103:todoitem
        uint32 style = 1;
        uint32 alignment = 2; // 0:left, 1:center, 2:right, 3:justified
        int32 indent = 4;
        Todo todo = 5;
    }
    message Font {
        string name = 1;
        float pointSize = 2;
        uint32 fontHints = 3;
    }
    message AttachmentInfo {
        string attachmentIdentifier = 1;
        string typeUTI = 2;
    }
    message Todo {
        bytes todoUUID = 1;
        bool done = 2;
    }
    message Color {
        float red = 1;
        float green = 2;
        float blue = 3;
        float alpha = 4;
    }

**Drawings**
This info isn’t strictly necessary.  For each drawing, you’ll find a rendering in `FallbackImages/UUID.jpg`. I was curious whether I could recover / backup the original vector data, so I came up with the following. (The root object here is `Drawing`.)


    message Drawing {
        int64 serializationVersion = 1;
        repeated bytes replicaUUIDs = 2;
        repeated StrokeID versionVector = 3;
        repeated Ink inks = 4;
        repeated Stroke strokes = 5;
        int64 orientation = 6;
        StrokeID orientationVersion = 7;
        Rectangle bounds = 8;
        bytes uuid = 9;
    }
    message Color {
        float red = 1;
        float green = 2;
        float blue = 3;
        float alpha = 4;
    }
    message Rectangle {
        float height = 4;
        float originX = 1;
        float originY = 2;
        float width = 3;
    }
    message Transform {
        float a = 1;
        float b = 2;
        float c = 3;
        float d = 4;
        float tx = 5;
        float ty = 6;
    }
    message Ink {
        Color color = 1;
        string identifier = 2;
        int64 version = 3;
    }
    message Stroke {
        int64 inkIndex = 3;
        int64 pointsCount = 4;
        bytes points = 5;
        Rectangle bounds = 6;
        bool hidden = 9;
        double timestamp = 11;
        bool createdWithFinger = 12;
        Transform transform = 10;
    }

The byte array `points` is compactly encoded list of points. It is a sequence of this struct:


    struct PKCompressedStrokePoint {
        float timestamp;            // timestamp (delta from somethin, probably previous)
        float xpos;
        float ypos;
        unsigned short radius;      // radius*10
        unsigned short aspectRatio; // aspectRatio*1000
        unsigned short edgeWidth;   // edgeWidth*10
        unsigned short force;       // force*1000
        unsigned short azimuth;     // azimuth*10430.2191955274
        unsigned char altitude;     // altitude*162.338041953733
        unsigned char opacity;      // opacity*255
    };

The `inkIndex` field points into the array `inks`. The `identifier` in an `Ink` includes stuff like `com.apple.ink.marker`. I kinda fudged this in my svg generation - apple’s rendering code is much more sophisticated, taking azimuth/altitude into account.  My code works well enough for pen, but falls short on marker.  You may be better off with the jpeg.

**Tables**
This one is complicated, and I think a bit of explanation is in order to explain why it’s structured this way. They are trying to model a table with multiple people editing at the same time. Editing the contents of a cell is essentially a solved problem (it’s complicated, but it’s solved above - each cell is its own "document” - a `String` object from above).

But in addition to this, people are adding, removing, and reordering columns.  You want to ensure that if two people move a column or one person adds a column and another adds a row, things end up in a sane state, no matter which order you see the operations.

To do this, we consider the rows to be an ordered set of uuids. (And the same for the columns.) Then you have a map of column guid → row guid → String object. 

The data itself a pile of CRDTs encoded with something like NSKeyedArchiver, but built on top of protobuf for these CRDTs. The root object contains a few tables, and a list of objects (like NSKeyedArchiver), and reverenced by index via a variant time that apple calls `ObjectID`. (I managed to figure this out by generically decoding the protobuf data and looking at it, but later found they ship an older revision of the proto files to their web app.)

This is the variant type used below:


    message ObjectID {
        uint64 unsignedIntegerValue = 2;
        string stringValue = 4;
        uint32 objectIndex = 6;
    }

The `objectIndex` is a index into the list of `object` in `Document`. 

The root `Document` is:


    message Document {   
        repeated DocObject object = 3;
        repeated string keyItem = 4;
        repeated string typeItem = 5;
        repeated bytes uuidItem = 6;
    }
    
    message DocObject {
        RegisterLatest registerLatest = 1;
        Dictionary dictionary = 6;
        String string = 10;  // this is our fancy String above
        CustomObject custom = 13;
        OrderedSet orderedSet = 16;    
    }
    

The first object in the `object` field is the root object. A `CustomObject` is essentially a key/value map with a type.  The keys are indexed from `keyItem` and type is from `typeItem`.


    message CustomObject {
        int32 type = 1; // index into "typeItem" below
        message MapEntry {
            required int32 key = 1; // index into keyItem below
            required ObjectID value = 2;
        }
        repeated MapEntry mapEntry = 3;
    }

For a UUID, the type is `com.apple.CRDT.NSUUID` and there is a `UUIDIndex` field whose value is the index of the UUID in `uuidItem` of the `Document`.

A `RegisterLatest` is just a CRDT for a value.  There is a clock, not shown here, which helps with merging conflicts.  It’s last write wins. It only appears to be used to point at an NSString custom object holding “CRTableColumnDirectionLeftToRight”.  I’m ignoring this at the moment.


    message RegisterLatest {
        ObjectID contents = 2;
    }

A `Dictionary` object holds object id for both key and value. In practice the keys points to a UUID `CustomObject` and the value is either a UUID or another dictionary. (This is a last-write-wins CRDT, I’m leaving out the clock values.) 


    message Dictionary {
        message Element {
           ObjectID key = 1;
           ObjectID value = 2;
        }
        repeated Element element = 1;
    }

Which leaves us with `OrderedSet`.  An ordered set leverages `String` to provide an vector of UUIDs via `TTArray`.  Pairs of string position to UUID are stored in `attachments` of `TTArray`.  Surrounding that is `Array` which has a CRDT Dictionary to map the uuid of the TTArray to the content in that position of the array (which happens to also be a UUID in this case).  And surrounding that is `OrderedSet` which contains another `Dictionary` of UUID to UUID, but the keys and values are the same (uuids from the TTArray space). This seems to be used to filter out deleted items in the case where you simultaneously move and delete a column. (The move does delete + add and the delete does a delete, so a copy remains in the Array.)

Two conflicting moves will also create duplicates in the array. Apple appears to handle this by ignoring all but the first instance. (And cleaning it up on the next move of that column.)


    message OrderedSet {
        Array ordering = 1;
        Dictionary elements = 2; // set of elements that haven't been deleted
    }
    message Array {
        TTArray array = 1; // TTArray
        Dictionary contents = 2; // map of TTArray uuid to content uuid
    }
    message TTArray {
        String contents = 1; // we don't actually reference this.
        ArrayAttachment attachments = 2; // list of (position -> uuid
    }
    message ArrayAttachment {
        int64 index = 1;
        bytes uuid = 2;
    }


**Decoding Tables**

Ok, so to decode a table, the root object will be a CustomObject with the fields:

| Field       | Value                                                       |
| ----------- | ----------------------------------------------------------- |
| crRows      | OrderedSet for row uuids                                    |
| crColumns   | OrderedSet for column uuids                                 |
| cellColumns | Dictionary of column uuid → Dictionary of row uuid → String |

The `value` of these fields are referenced by `objectIndex`.  Both `crRows` and `crColumns` are a `CustomObject` of type `OrderedSet`. 

To get the list of uuids for these ordered sets, take each `ordering.array.attachments.uuid`, filter out values that don’t appear as keys in `elements` , and look up each of the resulting values in the dictionary `ordering.contents`. 

Then iterate through the column uuids and look them up in cellColumns. The result, for each column, will be a dictionary.  In this dictionary, look up each row uuid.  The result, if present, will be a String object (or rather an ObjectId pointing to a String object).  This is the content of the table cell.



## TODO

I still need to write up the attachment, folder, and encryption stuff.


