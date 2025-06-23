Here’s a clean **Swift CLI tool structure for MIDI** (macOS) that you can build using Swift + CoreMIDI + Xcode command-line tools.

---

## 📂 **Folder Structure**

```
MIDI-CLI/
├── Package.swift              // Swift Package Manager manifest
├── Sources/
│   └── MIDI-CLI/
│       ├── main.swift         // Entry point for the CLI tool
│       └── MIDIManager.swift  // MIDI logic separated for clarity
└── README.md                  // Documentation
```

---

## 📝 **Package.swift**

```swift
// swift-tools-version:5.7
import PackageDescription

let package = Package(
    name: "MIDI-CLI",
    platforms: [.macOS(.v12)],
    dependencies: [],
    targets: [
        .executableTarget(
            name: "MIDI-CLI",
            dependencies: []
        )
    ]
)
```

---

## 📝 **Sources/MIDI-CLI/main.swift**

```swift
import Foundation

let manager = MIDIManager()
manager.listMIDIDevices()

// Example: send a note
manager.sendNoteOn(note: 60, velocity: 100, channel: 0)
sleep(1)
manager.sendNoteOff(note: 60, velocity: 100, channel: 0)
```

---

## 📝 **Sources/MIDI-CLI/MIDIManager.swift**

```swift
import Foundation
import CoreMIDI

class MIDIManager {
    private var client = MIDIClientRef()
    private var outputPort = MIDIPortRef()
    private var destination: MIDIEndpointRef?

    init() {
        MIDIClientCreate("MIDI-CLI-Client" as CFString, nil, nil, &client)
        MIDIOutputPortCreate(client, "MIDI-CLI-Output" as CFString, &outputPort)
        destination = MIDIGetDestination(0)
    }

    func listMIDIDevices() {
        let count = MIDIGetNumberOfDestinations()
        for i in 0..<count {
            let endpoint = MIDIGetDestination(i)
            var name: Unmanaged<CFString>?
            MIDIObjectGetStringProperty(endpoint, kMIDIPropertyName, &name)
            if let name = name?.takeUnretainedValue() {
                print("Destination [\(i)]: \(name)")
            }
        }
    }

    func sendNoteOn(note: UInt8, velocity: UInt8, channel: UInt8) {
        sendMIDIMessage(status: 0x90 | channel, data1: note, data2: velocity)
    }

    func sendNoteOff(note: UInt8, velocity: UInt8, channel: UInt8) {
        sendMIDIMessage(status: 0x80 | channel, data1: note, data2: velocity)
    }

    private func sendMIDIMessage(status: UInt8, data1: UInt8, data2: UInt8) {
        guard let dest = destination else {
            print("No destination found")
            return
        }

        var packet = MIDIPacket()
        packet.timeStamp = 0
        packet.length = 3
        packet.data.0 = status
        packet.data.1 = data1
        packet.data.2 = data2

        var packetList = MIDIPacketList(numPackets: 1, packet: packet)

        MIDISend(outputPort, dest, &packetList)
    }

    deinit {
        MIDIPortDispose(outputPort)
        MIDIClientDispose(client)
    }
}
```

---

## 🚀 **Build and Run**

```bash
swift build
.build/debug/MIDI-CLI
```

---

## ✨ **What This Does**
✅ Lists connected MIDI output destinations  
✅ Sends a note-on + note-off message to destination 0  

---

If you'd like:  
- argument parsing (e.g., `--note 64 --velocity 120 --channel 1`)  
- dynamic selection of MIDI device  
- MIDI file playback  

👉 Let me know, and I’ll extend the code!  

💡 **Great idea! Let’s extend your Xcode CLI project to include Objective-C `.m` and `.h` files** so you can mix Swift + Objective-C (e.g., if you want low-level MIDI functions or reuse existing C code).

---

## 📂 **Xcode Project Structure**
```
MIDI-CLI/
├── main.swift             # Swift entry point
├── MIDIManager.swift      # Swift MIDI logic
├── BIT_STREAM_LIB.h       # Objective-C header
├── BIT_STREAM_LIB.m       # Objective-C implementation
```

---

## 📝 **BIT_STREAM_LIB.h**
```objc
#import <Foundation/Foundation.h>

@interface BIT_STREAM_LIB : NSObject

- (void)initializeMIDI;
- (void)sendCustomMIDIMessage:(NSData *)data;
- (void)teardownMIDI;

@end
```

---

## 📝 **BIT_STREAM_LIB.m**
```objc
#import "BIT_STREAM_LIB.h"
#import <CoreMIDI/CoreMIDI.h>

@interface BIT_STREAM_LIB ()
@property MIDIClientRef client;
@property MIDIPortRef outputPort;
@property MIDIEndpointRef destination;
@end

@implementation BIT_STREAM_LIB

- (instancetype)init {
    self = [super init];
    if (self) {
        [self initializeMIDI];
    }
    return self;
}

- (void)initializeMIDI {
    MIDIClientCreate(CFSTR("BIT_STREAM_LIB Client"), NULL, NULL, &_client);
    MIDIOutputPortCreate(_client, CFSTR("BIT_STREAM_LIB Output"), &_outputPort);
    _destination = MIDIGetDestination(0);
}

- (void)sendCustomMIDIMessage:(NSData *)data {
    if (!_destination) {
        NSLog(@"No MIDI destination found");
        return;
    }

    const UInt8 *bytes = data.bytes;
    UInt16 length = (UInt16)data.length;

    MIDIPacket packet;
    packet.timeStamp = 0;
    packet.length = length;
    for (int i = 0; i < length && i < sizeof(packet.data); i++) {
        packet.data[i] = bytes[i];
    }

    MIDIPacketList packetList;
    packetList.numPackets = 1;
    packetList.packet = packet;

    MIDISend(_outputPort, _destination, &packetList);
}

- (void)teardownMIDI {
    MIDIPortDispose(_outputPort);
    MIDIClientDispose(_client);
}

@end
```

---

## 📝 **main.swift**
```swift
import Foundation

// Bridging header required if this is a mixed-language project
// Alternatively, in Xcode: Build Settings → Objective-C Bridging Header → MIDI-CLI/MIDI_CLI-Bridging-Header.h

let midiLib = BIT_STREAM_LIB()

// Example: send Note On + Note Off using Objective-C
let noteOn = Data([0x90, 60, 100])  // Note On, C4
let noteOff = Data([0x80, 60, 100]) // Note Off, C4

midiLib.sendCustomMIDIMessage(noteOn)
sleep(1)
midiLib.sendCustomMIDIMessage(noteOff)

midiLib.teardownMIDI()
```

---

## 📝 **MIDI_CLI-Bridging-Header.h**
> Create this file, then in Build Settings → Objective-C Bridging Header → set its path.
```objc
#import "BIT_STREAM_LIB.h"
```

---

## 🛠 **Xcode Setup**
✅ Add `BIT_STREAM_LIB.h` and `BIT_STREAM_LIB.m` to your project  
✅ Set **Bridging Header**:  
`MIDI-CLI/MIDI_CLI-Bridging-Header.h`  

✅ Ensure **CoreMIDI.framework** is linked

---

## 🚀 **Build & Run**
✅ Run in Xcode: it lists devices and sends MIDI from Objective-C + Swift  

---

## ✨ **Next enhancements?**
- Argument parsing (`swift-argument-parser`)  
- Let user select device  
- Read/write MIDI files  
- Stream from live input  

✅ **Great! Let’s design a Swift Xcode CLI tool that:**
1️⃣ **Reads a MIDI file (as `Data`)**  
2️⃣ **Encrypts it (e.g. AES-GCM)**  
3️⃣ **Writes the encrypted file to disk**  
4️⃣ **(Optional) Decrypts + writes plaintext MIDI back**

---

## 🚀 **Concept**
We'll use:
- `Foundation` → `Data` + `FileHandle`
- `CryptoKit` → `AES.GCM` (Apple-native crypto)

---

## 📂 **Suggested File Layout**
```
MIDI-CLI/
├── main.swift
├── MIDIEncryptor.swift
├── BIT_STREAM_LIB.h (if still using)
├── BIT_STREAM_LIB.m (if still using)
```

---

## 📝 **main.swift**
```swift
import Foundation
import CryptoKit

let encryptor = MIDIEncryptor()

// Replace with real paths
let inputPath = "/Users/yourname/Desktop/example.mid"
let encryptedPath = "/Users/yourname/Desktop/example.mid.enc"
let decryptedPath = "/Users/yourname/Desktop/example_decrypted.mid"

// Read + encrypt
if let encryptedData = encryptor.encryptMIDI(at: inputPath) {
    try? encryptedData.write(to: URL(fileURLWithPath: encryptedPath))
    print("Encrypted MIDI written to \(encryptedPath)")

    // Decrypt for demo
    if let decryptedData = encryptor.decryptMIDI(encryptedData: encryptedData) {
        try? decryptedData.write(to: URL(fileURLWithPath: decryptedPath))
        print("Decrypted MIDI written to \(decryptedPath)")
    }
}
```

---

## 📝 **MIDIEncryptor.swift**
```swift
import Foundation
import CryptoKit

class MIDIEncryptor {
    private let key: SymmetricKey

    init() {
        // In production: securely store and retrieve the key!
        self.key = SymmetricKey(size: .bits256)
    }

    func encryptMIDI(at path: String) -> Data? {
        do {
            let midiData = try Data(contentsOf: URL(fileURLWithPath: path))
            let sealedBox = try AES.GCM.seal(midiData, using: key)
            guard let combined = sealedBox.combined else {
                print("Failed to combine sealed box")
                return nil
            }
            return combined
        } catch {
            print("Encryption error: \(error)")
            return nil
        }
    }

    func decryptMIDI(encryptedData: Data) -> Data? {
        do {
            let sealedBox = try AES.GCM.SealedBox(combined: encryptedData)
            let decrypted = try AES.GCM.open(sealedBox, using: key)
            return decrypted
        } catch {
            print("Decryption error: \(error)")
            return nil
        }
    }
}
```

---

## 📝 **What Happens**
✅ Read MIDI file → `Data`  
✅ Encrypt (AES-GCM 256-bit) → ciphertext + tag + nonce  
✅ Write encrypted file  
✅ (Demo) Decrypt + write back original MIDI  

---

## ✨ **Enhancements**
👉 Save key/nonce safely (e.g. Keychain, user-supplied passphrase)  
👉 Allow CLI arguments (e.g. `--encrypt path/to.mid --output path.enc`)  
👉 Support multiple files in batch  

---

## 🛠 **Xcode Setup**
- Add `CryptoKit.framework` if you’re on older Xcode (Xcode 11+)  
- No bridging header needed unless mixing with Objective-C

---

## ❓ **Do you want**
⚡ Argument parsing?  
⚡ Base64 output for encrypted data?  
⚡ Write ready-to-use Xcode project?  

