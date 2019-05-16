### Serialize Struct:

Ar &lt;&lt; ObjectClass;

uStruct-&gt;SerializeItem(Ar, Allocation, nullptr);

if (Ar.ArIsLoading)  
Object = NewObject&lt;UObject&gt;(ObjectClass, ...);

if (Ar.WantBinaryPropertySerialization())  
ObjectClass-&gt;SerializeBin(Ar, Object);  
else  
ObjectClass-&gt;SerializeTaggedProperties(Ar,Object,...);

**Serialize Enum:**

FORCEINLINE friend FArchive& operator&lt;&lt;(FArchive& Ar, EnumType& Value)

> {
>
> return Ar &lt;&lt; (\_\_underlying_type(EnumType)&)Value;
>
> }

**UObject**

ObjectReader/ObjectWriter only to be able to serialize the actual objects into byte arrays. We have not found any other archives that do this for us in a good way.

_From &lt;<https://udn.unrealengine.com/questions/299982/serialize-objects-for-loadsave.html>&gt;_

These will iterate through the UProperties in the class and either write binary (fast but difficult or impossible to save delta properties during development) or tagged properties (slower but allows delta properties). So the idea is to first serialize the object class using Ar &lt;&lt; ObjectClass; then spawn an instance of that class (if loading), then serialize the properties using the class and instance.

_From &lt;<https://forums.unrealengine.com/development-discussion/c-gameplay-programming/1374656-how-to-load-an-object-from-binary-without-knowing-its-exact-class>&gt;_

### ArIsSaveGame = True

the super undocumented <span class="underline">secret to serializing any data to or from an object based on whether the CPF_SaveGame flag is set</span>, is super simple: When you are creating your FMemoryWriter, you MUST have the ArIsSaveGame flag set to true. By default, it's false. IF you forget to set this flag, the Memory Writer will try to serialize your ENTIRE object, and it will loop through EVERY property and try to serialize it.

_From &lt;<https://forums.unrealengine.com/development-discussion/c-gameplay-programming/88477-spawning-actors-from-serialized-data?116235-Spawning-Actors-from-Serialized-Data=&viewfull=1>&gt;_

**Saving data to file**

**Archive to bytearray to File**

<table><tbody><tr class="odd"><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p></td><td><p>FActorSaveData ActorRecord;</p><p>ActorRecord.ActorName = FName(*Actor-&gt;GetName());</p><p>ActorRecord.ActorClass = Actor-&gt;GetClass()-&gt;GetPathName();</p><p>ActorRecord.ActorTransform = Actor-&gt;GetTransform();</p><p> </p><p>FMemoryWriter MemoryWriter(ActorRecord.ActorData, true);</p><p>FSaveGameArchive Ar(MemoryWriter);</p><p>Actor-&gt;Serialize(Ar);</p><p> </p><p>SavedActors.Add(ActorRecord);</p><p>ISaveableActorInterface::Execute_ActorSaveDataSaved(Actor);</p></td></tr></tbody></table>

After all the actor records are saved, we can create the final save game.

<table><tbody><tr class="odd"><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p></td><td><p>FSaveGameData SaveGameData;</p><p> </p><p>SaveGameData.GameID = "1234";</p><p>SaveGameData.Timestamp = FDateTime::Now();</p><p>SaveGameData.SavedActors = SavedActors;</p><p> </p><p>FBufferArchive BinaryData;</p><p> </p><p>BinaryData &lt;&lt; SaveGameData;</p></td></tr></tbody></table>

After we have created the final save file and serialized it, we can store the byte array to a file.

<table><tbody><tr class="odd"><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p></td><td><p>if (FFileHelper::SaveArrayToFile(BinaryData, *FString("TestSave.sav")))</p><p>{</p><p>UE_LOG(LogTemp, Warning, TEXT("Save Success! %s"), FPlatformProcess::BaseDir());</p><p>}</p><p>else</p><p>{</p><p>UE_LOG(LogTemp, Warning, TEXT("Save Failed!"));</p><p>}</p><p> </p><p>BinaryData.FlushCache();</p><p>BinaryData.Empty();</p></td></tr></tbody></table>

_From &lt;<http://runedegroot.com/saving-and-loading-actor-data-in-unreal-engine-4/>&gt;_

**Alternate way to do it directly with proxy archive:**

FArchive\* FileWriter = IFileManager::Get().CreateFileWriter(\*ProfileFileName);

if(FileWriter != nullptr)

{

FCollisionAnalyzerProxyArchive Ar(\*FileWriter);

int32 Magic = COLLISION_ANALYZER_MAGIC;

int32 Version = COLLISION_ANALYZER_VERSION;

Ar << Magic;

Ar << Version;

Ar << Queries;

FileWriter->Close();

delete FileWriter;

FileWriter = NULL;

UE_LOG(LogCollisionAnalyzer, Log, TEXT("Saved collision analyzer data to file '%s'."), \*ProfileFileName);

}

### **Loading Actors**

**Loading the data from file**

To load the binary data

<table><tbody><tr class="odd"><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p></td><td><p>TArray&lt;uint8&gt; BinaryData;</p><p>if (!FFileHelper::LoadFileToArray(BinaryData, *FString("TestSave.sav")))</p><p>{</p><p>UE_LOG(LogTemp, Warning, TEXT("Load Failed!"));</p><p>return;</p><p>}</p><p>else</p><p>{</p><p>UE_LOG(LogTemp, Warning, TEXT("Load Succeeded!"));</p><p>}</p></td></tr></tbody></table>

##### **Extracting data from binary**

<table><tbody><tr class="odd"><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p></td><td><p>FMemoryReader FromBinary = FMemoryReader(BinaryData, true);</p><p>FromBinary.Seek(0);</p><p> </p><p>FSaveGameData SaveGameData;</p><p>FromBinary &lt;&lt; SaveGameData;</p><p> </p><p>FromBinary.FlushCache();</p><p>BinaryData.Empty();</p><p>FromBinary.Close();</p></td></tr></tbody></table>

_From &lt;<http://runedegroot.com/saving-and-loading-actor-data-in-unreal-engine-4/>&gt;_

### Serialize Objects & Names as strings:

struct FSaveGameArchive : public FObjectAndNameAsStringProxyArchive

{

FSaveGameArchive(FArchive& InInnerArchive)

: FObjectAndNameAsStringProxyArchive(InInnerArchive, true)

{

ArIsSaveGame = true;

}

};

_From &lt;<http://runedegroot.com/saving-and-loading-actor-data-in-unreal-engine-4/>&gt;_

### PostFixupReferences

- Fixup actor/object references after serialization

_From &lt;<https://forums.unrealengine.com/development-discussion/c-gameplay-programming/88477-spawning-actors-from-serialized-data?116235-Spawning-Actors-from-Serialized-Data=&viewfull=1>&gt;_