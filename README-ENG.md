#GOPRO.PY
This file contains a set of functions and procedures, created specifically for managing the operation of a GoPro camera.

These methods are later called from other containers by specialized ROS-requests.



The main method in the module is:

 ```def handle_record(msg)``` - This method is used to run the basic procedure for writing and publishing the files to IPFS network.

In the beginning of the method, we check whether the variable msg.data contains the value or not.

```Python
...

if msg.data:
            # Start recording the video
            try:
                recording(True)
                return SetBoolResponse(True, '')
            except:
                return SetBoolResponse(False, 'Unable to start recording')

...
```

Then we command to start or stop recording on a GoPro camera. Note that the camera recording starts automatically, without any additional actions.
###### Method ```recording()``` will be described below. 



Next, let's see what happens when you stop recording:

```Python
...

else:
            # Stop recording and publish thumbnail & video files
            try:
                recording(False)
            except Exception as e:
                return SetBoolResponse(False, 'Unable to stop recording: {0}'.format(e))

            thumbhash = ''
            try:
                time.sleep(2)
                m = list(medias())[-1]
                msg = String()

                thumbhash = ipfsPublish(getThumb(m))
                msg.data = thumbhash
                thumbnail.publish(msg)

                videohash = ipfsPublish(getVideo(m))
                msg.data = videohash
                video.publish(msg)
            except Exception as e:
                return SetBoolResponse(False, 'Unable to publish media: {0}'.format(e))

            return SetBoolResponse(True, videohash)

...
```
This code can be broken down into the following set of methods:



 - ```recording()``` - This method is responsible for two actions: removing all files from the camera and controlling the shutter of the camera.
 - - Let's look at this module:
 
 ```Python
 def recording(enable):
    rospy.loginfo('Set video enable {0}'.format(enable))
    camera = GoProHero(password='password')
    try:
       if enable:
         urlopen('http://10.5.5.9/gp/gpControl/command/storage/delete/all').read()
         time.sleep(5)
    except:
       rospy.loginfo('FORMATTING FAIL')
    camera.command('record', 'on' if enable else 'off')
 ```
 
Note that in the beginning we create an object `` `camera = GoProHero (password = 'password')` ``, which has a set of methods from the GoPro API. Next, we remove all the files from the camera to eliminate further difficulties when working with the GoPro file system.

``` Python
 urlopen('http://10.5.5.9/gp/gpControl/command/storage/delete/all').read()
 time.sleep(5)
```  
 
Note that after the deletion, we call the `` `sleep``` method to wait for the camera to format.

And then we call the method ```camera.command()```

``` Python
 camera.command('record', 'on' if enable else 'off')
``` 



 - ```medias()``` - This method is designed to return a list of all media files that are on the camera.
 - - Let's look at this module:

 ```Python
 def medias():
    '''
        Get media list by GoPro HTTP API
    '''
    url ='http://10.5.5.9:8080/gp/gpMediaList'
    media = json.loads(urlopen(url).read())
    for m in media['media']:
        for n in m['fs']:
            rospy.loginfo('Acuired media {0}'.format(m['d']))
            yield '{0}/{1}'.format(m['d'], n['n'])
```

This method decrypts the JSON file received from the camera. It contains the structure of the file system with the correct location of the GoPro multimedia files. The method is a generator.



 - ```getVideo()```
 - -  Let's look at this module:
 
 ```Python
 def getVideo(media):
    '''
        Get video by GoPro
    '''
    rospy.loginfo('Get video of {0}'.format(media))
    url = 'http://10.5.5.9:8080/videos/DCIM/{0}'.format(media)
    video = urlopen(url)
    filename = 'test.mp4'
    with open(filename,'wb') as output:
        output.write(video.read())
    return filename
 ```
 
This method downloads a file from the camera and returns its name.



 - ```ipfsPublish()``` - This method publishes the saved file to the Interplanetary File System (IPFS).
 
 - - Let's look at this module:
 
```Python
def ipfsPublish(data):
    '''
        Publish bytes by IPFS client
    '''
    rospy.loginfo('IPFS publish data length {0}'.format(len(data)))
    return ipfsapi.connect('127.0.0.1', 5001).add(data)['Hash']
```

Note that this method uses `` `IPFS_API``` and publishes the previously downloaded file and returns the hash of this file, which is part of the link to` `http: // ipfs.io / ipfs /` ``

Uploading is relatively fast, as the file will be immediately available for download.
