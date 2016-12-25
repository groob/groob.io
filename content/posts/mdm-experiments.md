+++
date = "2015-11-14T00:49:35-05:00"
title = "MDM from scratch"

+++

This summer, at PSU MacAdmins, [Pepijn Bruienne](https://twitter.com/bruienne) showcased project-imas/mdm-server, a simple implementation of a iOS MDM server. I had little interest in an MDM so far, because of it's limited applications on OS X. With Apple's DEP program, and recent additions such as OS X Software Update management through MDM, the whole idea became more appealing. 

I've been reading some of the Apple Developer docs to get a better understanding of how the MDM protocol works behind the scenes, and implementing my own server. Here are some of my notes(with some Go code) about the very basics of MDM.

# Push Certificate
Getting the certificates correct, was by far the most laborious part of setting up an MDM server. 
First, I had to get a Push Certificate - which requires an enterprise account with Apple. See [steps 1-8](https://github.com/project-imas/mdm-server/blob/master/README.md#setup) on the project-imas mdm-server readme.

The Push certificate is required to communicate with the Apple Push Notification Service(APNS). Beside the APNS, we also need valid TLS certificates for our MDM server. We also need an identity certificate, which will be deployed to our devices. 
See step 9 of the project imas readme. 

# Enrollment profile. 
Our next step is to create an enrolment profile for our devices. I started by downloading a profile from the OS X Server Profile Manager and removing all settings. Using Configurator 2 utility, I added the `Identity.p12` certificate from the previous step. 

Ideally, that should create a valid profile, but I kept seeing an error message that the Identity certificate was not found. I had to open the `.mobileconfig` file in a text editor and make sure that the `IdentityCertificateUUID` key had the same value as the `PayloadUUID` of the identity cert. Once I did that, both iOS and OS X Devices enrolled without any issues. 

# Server
The MDM server, is a web server that listens for devices to check in, and sends back various XML payloads with instructions/configurations that the enrolled devices then execute. That's mostly all it does, but let's explore this server/device interaction in detail. 

## Checking in
In our enrollment profile, one of the keys we had to specify was a checkin URL. I chose /checkin but this could be anything.
```xml
            <key>CheckInURL</key>
            <string>https://my.mdm.co/checkin</string>
```

This is the endpoint where our devices will be sending us information about enrollment, and also when the configuration profiles are removed. 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
        <key>AwaitingConfiguration</key>
        <false/>
        <key>MessageType</key>
        <string>TokenUpdate</string>
        <key>PushMagic</key>
        <string>10B5E72B-ACB6-40DE-DFE2-3051F5B407EE</string>
        <key>Token</key>
        <data>
        a68O3jkya2ToAqkkNyouokBI+9CITIkLbY1Q1ySx8ls=
        </data>
        <key>Topic</key>
        <string>com.apple.mgmt.External.7faed215-f838-4618-a210-efhhhh861541</string>
        <key>UDID</key>
        <string>601E935B-2F63-5AC1-BDB2-C9EB0B72B53D</string>
</dict>
</plist>
```

The information we really care about in this checkin are the Token(a base64 encoded string) and the "PushMagic" string.We'll use both of these, as well as the UDID to communicate with our devices. 
Now that the device has checked in with our server, we might want to ask it to check in again. To do that, we will take the PushMagic string and send the following JSON payload over the APNS. `{"mdm":"10B5E72B-ACB6-40DE-DFE2-3051F5B407EE"}`

By sending this special push notification message to the device, we're asking it to report back to the MDM server, so that we can issue our next command. 

## APNS
There are quite a few libraries out there for APNS because it's the same protocol as push notifications for iOS apps. 
For my purposes, I used  a [fork](https://github.com/nicolasgomollon/apns/network) of timehop's apns Go library.

I learned quite a bit in the process of getting APNS to work, including that there are some issues with the current [protocol](http://redth.codes/the-problem-with-apples-push-notification-ser/) and that apple will be releasing an http2 based protocol for push notifications [soon](https://developer.apple.com/videos/play/wwdc2015-720/). 

```Go
func push()
    // https://github.com/joekarl/go-libapns#pem-certs
    c, err := apns.NewClientWithFiles(apns.ProductionGateway, "cert.pem", "key-noenc.pem")
    if err != nil {
        log.Fatal("could not create new client", err.Error())
    }

    go func() {
        for f := range c.FailedNotifs {
            fmt.Println("Notif", f.Notif.ID, "failed with", f.Err.Error())
        }
    }()

    p := apns.NewPayload()
    p.MDM = "10B5E72B-ACB6-40DE-DFE2-3051F5B407EE"

    m := apns.NewNotification()
    m.Payload = p

    // the library expects DeviceToken to be hex encoded, but apple recently
    // changed the format to Base64 encoded, so we must decode/encode before proceeding here.

    tokenData := "a68O3jkya2ToAqkkNyouokBI+9CITIkLbY1Q1ySx8ls="
    hextoken, err := base64.StdEncoding.DecodeString(tokenData)
    if err != nil {
        log.Fatal(err)
    }
    HextTokenStr := hex.EncodeToString(hextoken)
    m.DeviceToken = HextTokenStr

    m.Priority = apns.PriorityImmediate
    m.Identifier = 12513

    c.Send(m)
    // Wait for all notifications to be pushed before exiting.
    // http://redth.codes/the-problem-with-apples-push-notification-ser/
    for (c.Sent + c.Failed) < c.Len {
        time.Sleep(30 * time.Second)
    }
}
```

## MDM Payloads

Once we sent out a push notification, the device should contact our MDM server over https. This time however, instead of using the `/checkin` endpoint, it will connect to the ServerURL endpoint, which we also define in our enrollment profile.
```xml
            <key>ServerURL</key>
            <string>https://my.mdm.co/connect</string>
```

The device will send another plist, with a status update and wait for the server to respond. It's possible to get a different kind of status, like `NotNow`, but so far Idle looks most common. 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
        <key>Status</key>
        <string>Idle</string>
        <key>UDID</key>
        <string>50233a56cbec8492c0967c10ff4dbf887d864858</string>
</dict>
</plist>
```

Now we can send something useful to the device. We can request information from the device, or tell it to perform an action - like installing a configuration profile, installing an app or trigger an OS update. The server sends a plist, which contains a `CommandUUID` key which identifies the command, and a dict with a `RequestType` key, and the fields necessary for that request. 
For example, to install a profile, we'd set the RequestType to `InstallProvisioningProfile` and then add a `<data>` field containing a base64 encoded mobileconfig file. 

I started by exploring something more harmless - the `DeviceInformation` command.  

The payload structure:

```Go
package mdm 
type deviceInformation struct {
    Queries []string
}

type command struct {
    RequestType string
    deviceInformation
}

type Payload struct {
    CommandUUID string
    Command     *command
}

func NewCommand(cmdType string) *command {
    cmd := &command{}
    switch cmdType {
    case "DeviceInformation":
        cmd.RequestType = "DeviceInformation"
        cmd.Queries = []string{"LastCloudBackupDate"}
    }
    return cmd
}
```

And the HTTPS Server.

```Go

func infoHandler(w http.ResponseWriter, r *http.Request) {
    // create new MDM Command
    cmd := mdm.NewCommand("DeviceInformation")
    payload := &mdm.Payload{
        // should be a random UUID here, but we're just testing
        CommandUUID: "9F09D114-BCFD-42AD-A974-371AA7D6256E",
        Command:     cmd,
    }
    // create and send the plist to the device.
    err := plist.NewEncoder(w).Encode(payload)
    if err != nil {
        log.Fatal(err)
    }
}


func main() {
    http.HandleFunc("/connect", infoHandler)
    log.Fatal(http.ListenAndServeTLS(":443", "mdmserver.crt", "Server.key", nil))
}
```

The Device responded back with the informatin I was looking for - the LastCloudBackupDate.
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
        <key>CommandUUID</key>
        <string>9F09D114-BCFD-42AD-A974-371AA7D6256E</string>
        <key>QueryResponses</key>
        <dict>
                <key>LastCloudBackupDate</key>
                <date>2015-11-14T11:59:40Z</date>
        </dict>
        <key>Status</key>
        <string>Acknowledged</string>
        <key>UDID</key>
        <string>50233a56cbec8492c0967c10ff4dbf887d864858</string>
</dict>
</plist>
```

This is about as far as I've explored the MDM, although there is not too much left. We have to write and test all the possible responses, and decide how to handle each response. 
