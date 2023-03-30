# Tuvali implementation
Before we understand how Tuvali component performs activities on the BLE layer to transfer VC/VP. We can take a look at how BLE communication works in general between two devices A & B.

## How does BLE Communication work?
![BLE Communication](../_images/ble-high-level-comm-generic.png)

In BLE communication, one device is designated as Peripheral and another one designated as Central.

Peripheral can perform advertisement which can be read by a Central device and connect to it, if the advertisement is connectable.

Once the connection is made, Central can perform further actions on the device like 
- discovering services and characteristics
- subscribing to notifications on characteristics
- write/read data from characteristics
- disconnect from device

While peripheral can perform actions like 
- sending notifications to subscribed characteristics
- respond to read requests from Central

The above diagram explains the sequence of actions for a BLE communication in general.

1. Advertisement from Peripheral
2. Connection establishment & additional data exchange
3. Service & Characteristic discovery from Central
4. Characteristic subscription on Peripheral
5. Write with response to Characteristic
6. Write without response to characteristic
7. Send Notification from GATT Server
8. Disconnection from GATT Server/Client

More details about other BLE terminology used here can be found in standard BLE specifications of 4.2 and above.

## How Tuvali works with BLE to transfer VC from Central to Peripheral
![OpenID for VP over BLE Implementation](../_images/openid-for-vp-over-ble-spec-impl.png)

> Note: Tuvali is supposed to implement OpenID for VP over BLE specification. As part of it, both VP request and response transfer should be implemented. However the current version of Tuvali only transfers VC from Central to Peripheral.

In case of Tuvali, the entire VC transfer flow can be divided into following stages
1. Connection Setup & Cryptographic key exchange
2. Data transfer
3. Connection Closure

### 1. Connection Setup & Cryptographic key exchange
Steps 1 to 6 mentioned in the above diagram explains how first stage is achieved.
Before even the advertisement is started, peripheral generates a 32 byte public key. This public key is sent to Central as two parts. First part will have 5 bytes of public key sent as part of the advertisement payload and while second part is sent as part of SCAN_RESP.

Since the advertisement from Peripheral is of connectable type, Central would send out a SCAN_REQ on receiving advertisement and gets the remaining 27 bytes of peripheral public key. 
Post that, Central would derive a public key pair and send the its 32 bytes public key on Identify characteristic of Peripheral.

Once the public keys are exchanged between Central and Peripheral, a set of keys are derived on both sides which would be used for encryption and decryption of data on the wire.

> Cryptographic Algorithm usage:  
> - Ephemeral Key Pair is generated from X25519 curve
> - Key Agreement uses ECKA-DH (Elliptic Curve Key Agreement Algorithm – Diffie-Hellman) as defined in BSI TR-03111
> - Wallet and Verifier derives respective keys using HKDF as defined in RFC5869
> - Encryption/Decryption uses AES-256-GCM (192) (GCM: Galois Counter Mode)  as defined in NIST SP 800-38D


### 2. Data transfer
Steps 7 to 11 mentioned in the above diagram explains how second stage is achieved. 
Before VC is transferred, Central performs encryption and compression and communicate the resultant data size by writing to Response Size characteristic to Peripheral. The actual data transfer would happen on `Submit Response` characteristic.

Since the maximum allowed write value for a characteristic is limited to 512 bytes. The actual VC data is split by a component called Chunker into multiple smaller chunks. After the split, the data is transferred on the `Submit response`characteristics one after another until all chunks are completely sent.

Peripheral on the other hand, receives data on the `Submit response` characteristic. The received chunks are collected and final VC is assembled by a component called Assembler.

At the end of sending one frame of data from Central. It would request for a transfer report via `Transfer report request` characteristic. Peripheral responds with a summary of missing/corrupted chunks sequence numbers via another `Transfer report summary` characteristic. 

Central would read the Transfer report sumamry to understand if the Peripheral received all the chunks properly. If summary report has atleast one chunk sequence number. Central would send those specific chunks to Peripheral which is called Failure frame.

The failure frame will be sent from Central repeatedly until Transfer report summary is successful. If during the process, Central reached the maximum allowed failure frame retry limit, the tranfer is halted, devices will be disconnected and an error is generated (Please refer to API documentation on how this error can be read).

> - Gzip is the Compression algorithm used with default compression level
> - Each chunk have 2 bytes of CRC-16/Kermit added. Parameters for the same are: width=16 poly=0x1021 init=0x0000 refin=true refout=true xorout=0x0000 check=0x2189 residue=0x0000 name="CRC-16/KERMIT"


### 3. Connection closure
#### Disconnect initiated by Peripheral:
- On a sucessful data transfer 
- On non recoverable error occured on Peripheral

Peripheral notifies Central to perform disconnection via `Disconnect` characteristic mentioned in Step 12 of the above diagram.

#### Disconnect initiated by Central:
Central also performs disconnect in the following scenarios
- On a succesful data transfer
- Non recoverable error on Central
- Peripheral is out of range/disconnected
- Destroy Connection API

As part of connection closure, both central and peripheral cleans the held resources, cryptographic keys, bluetooth resources, to ensure that the subsequent transfer happens smoothly.



## Error Scenarios:

### Scenario 1: The verifier receives a `Failed to transfer` message and wallet receives a `Disconnected` message on the screen.
**Possible error scenarios:**
-   During VC transfer, if there is a failure in transfer of more than 70% of the data we get an exception on the verifier side and it disconnects from the wallet.
-   After the verifier and wallet establishes connection, the wallet initiates a MTU negotiation with the verifier. If the negotiated MTU is less than 64 Bytes, then the verifier throws an exception and disconnects from the wallet.
-   If the verifier receives the size of the VC as 0, it raises an exception and disconnects from the wallet.

### Scenario 2: The wallet receives a `Failed to transfer` message and verifier receives a `Disconnected` message on the screen.
**Possible error scenarios:**

-   After the verifier and wallet establishes connection, the wallet initiates a MTU negotiation with the verifier. If the wallet is unable to negotiate the MTU with the verifier, it raises an exception and disconnects from the verifier.
-   During VC transfer, if the transfer cannot be completed within the specified limit of retries, the wallet raises an exception and disconnects from the verifier.
-   After the wallet sends all the chunks, it requests for a transfer report. If the wallet is not able to send the request, it raises an exception and disconnects from the verifier.

Below are the exception message and the disconneect message which appears on the screen during error.

![Exception Message](../_images/exception-message.png)
> This message is displayed on the device throwing the exception.

![Disconnect Message](../_images/disconnect-message.png)
>This message is displayed whenever a device gets disconnected.

## Retry Scenarios:

-   **Backoff Strategy**: Exponential Backoff is a technique that retries a failing operation, with an exponentially increasing wait time, up to a maximum retry count(MAX_RETRY_LIMIT) or maximum backoff time(MAX_ELAPSE_TIME).

    Initially, it starts with 2 ms as wait time (INITIAL_WAIT_TIME) and increases exponentially with each failure until the count reaches to MAX_EXPONENT after which the wait time becomes constant (INITIAL_WAIT_TIME ^ MAX_EXPONENT).

    -   **BLE Service Discovery**: After the connection is established, the central attempts to discover all the services hosted by the peripheral. If it fails to discover our service, then the exponential backoff based retry will kick in.
        Here are the values:
        - MAX_RETRY_LIMIT  is 5 for android and 10 for IOS
        - MAX_ELAPSE_TIME is 100 ms
        - MAX_EXPONENT is 5

    -   **Request Transfer Report** [only for iOS]:  After the wallet writes all the chunks to the verifier, it requests for the transfer report. If the transfer report request write fails, then the exponential backoff based retry will kick in.
        Here are the values:
        - MAX_RETRY_LIMIT  is 5
        - MAX_ELAPSE_TIME is 100 ms
        - MAX_EXPONENT is 5

-   **Dynamic MTU negotiation**:

    -   **Android**: After the connection is established, the wallet initiates a MTU negotiation with an initial value of 512 bytes. If it fails, it retries with 185 and 100 bytes subsequently with a wait time of 500 ms each. If the negotiation fails after all retries, it throws an exception and disconnects from the wallet.

    -   **iOS**: iOS kicks off an MTU exchange automatically upon connection


## Constants:

- MAX_ALLOWED_DATA_LEN(509 Bytes): Maximum data length allowed for one write for both wallet and verifier

- MIN_MTU_REQUIRED(64 Bytes): Minimum bytes required to share public key transfer of wallet is 46. In order not to operate in edges, we chose the nearest value in power of 2 i.e. 64.

- MAX_FAILURE_FRAME_RETRY_LIMIT(15): Maximum limit to retry sending failure chunks to the verifier

## Characteristics UUID:


- IDENTIFY_REQUEST_CHAR_UUID (00002030-0000-1000-8000-00805f9b34fb): Characteristic for sending public key of wallet

- RESPONSE_SIZE_CHAR_UUID (00002033-0000-1000-8000-00805f9b34fb): Characteristic for sending VC size to the verifier

- SUBMIT_RESPONSE_CHAR_UUID (00002034-0000-1000-8000-00805f9b34fb): Characteristic for sending the entire VC

- TRANSFER_REPORT_REQUEST_CHAR_UUID (00002035-0000-1000-8000-00805f9b34fb): Characteristic for requesting for transfer report from the verifier

- TRANSFER_REPORT_RESPONSE_CHAR_UUID (00002036-0000-1000-8000-00805f9b34fb): Characteristic for sending transfer report to the wallet

- VERIFICATION_STATUS_CHAR_UUID (00002037-0000-1000-8000-00805f9b34fb): Characteristic for informing the wallet if the VC is accepted or rejected

- DISCONNECT_CHAR_UUID (00002038-0000-1000-8000-00805f9b34fb): Characteristic for notifying wallet to initiate the disconnection between the devices

## Service UUID:
-  SERVICE_UUID (0000AB29-0000-1000-8000-00805f9b34fb): Service UUID of the verifier

-  SCAN_RESPONSE_SERVICE_UUID (0000AB2A-0000-1000-8000-00805f9b34fb): Service UUID for uniquely identifying the scan response data to the wallet's SCAN_REQ
    

