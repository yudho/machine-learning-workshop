---


---

<h1 id="lab-1-person-identification-and-sentiment-analysis">Lab 1: Person Identification and Sentiment Analysis</h1>
<p>In this lab, we would utilize Amazon Rekognition to identify the person in front of a camera, that has been previously registered. In addition to that, we will stream the identified person’s emotions data to DynamoDB and analyze it. The analysis includes the person’s mood for the day and his/her mood history when captured in the camera.</p>
<h2 id="step-1--setup-identity-using-cognito">Step 1 : Setup Identity using Cognito</h2>
<p><strong>Note: You can skip this step if you have done it in other lab. Make sure you have the Identity Pool ID</strong><br>
The simple web application that we will build needs access to use AWS. Amazon Cognito provides an easy identity management to allow our JavaScript in HTML page assume IAM (Identity and Access Management) role for authorization to use AWS services.<br>
You can use Cognito for web and mobile app too.</p>
<p>Let’s create Cognito Identity Pool</p>
<p><strong>1.1.</strong> Login to your own AWS account or AWS account provided in the workshop<br>
<strong>1.2.</strong> Using your browser, go to Amazon Cognito US N.Virginia region via <a href="https://console.aws.amazon.com/cognito/home?region=us-east-1">https://console.aws.amazon.com/cognito/home?region=us-east-1</a><br>
<strong>1.3.</strong> Click “Manage Identity Pools” blue button.<br>
<strong>1.4.</strong> For Identity pool name, type: <strong>awsworkshop</strong><br>
<strong>1.5.</strong> Under “Unauthenticated identities” section, <strong>check</strong> “Enable access to unauthenticated identities”<br>
<strong>1.6.</strong> Click “Create Pool” blue button<br>
<strong>1.7.</strong> On next page, expand “View Details” section<br>
<strong>1.8.</strong> Leave the first role (authenticated role) as is. We use unauthenticated identity as for the workshop purpose, so that users do not need to login.<br>
<strong>1.9.</strong> For second role (role name = “Cognito_awsworkshopUnauth_Role”), expand “View Policy Document” section<br>
<strong>1.10.</strong> Click “Edit” link next to the policy, and click “Ok” when prompted. If you want to, you can read documentation first.<br>
<strong>1.11.</strong> Replace the policy with the following policy document to give it permission to access Amazon Rekognition.</p>
<pre><code>{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "mobileanalytics:PutEvents",
                "rekognition:*",
                "cognito-sync:*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "dynamodb:PutItem",
                "dynamodb:Query"
            ],
            "Resource": "arn:aws:dynamodb:us-east-1:*:table/People"
        }
    ]
}
</code></pre>
<p><strong>1.12.</strong> Click “Allow” button’<br>
<strong>1.13.</strong> On the next page (“Getting started with Amazon Cognito” page), change the “Platform” dropdown to <strong>JavaScript</strong><br>
<strong>1.14.</strong> Copy the code inside “Get AWS Credentials” box to a temporary txt file.</p>
<h2 id="step-2--build-the-face-indexing-web-application">Step 2 : Build the Face Indexing Web Application</h2>
<p>After authorization is setup, now we can build the web application. We will first build a single page HTML app to allow administrators to register persons using their face.<br>
<strong>2.1.</strong> On your laptop, make a new file named <strong>register.html</strong><br>
<strong>2.2.</strong> Paste the following code</p>
<pre><code>&lt;!doctype html&gt;
&lt;html lang="en"&gt;
    &lt;head&gt;
        &lt;script src="https://code.jquery.com/jquery-3.4.1.min.js" integrity="sha256-CSXorXvZcTkaix6Yvo6HppcZGetbYMGWSFlBw8HfCJo=" crossorigin="anonymous"&gt;&lt;/script&gt;
        &lt;link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap.min.css" integrity="sha384-HSMxcRTRxnN+Bdg0JdbxYKrThecOKuH5zCYotlSAcp1+c8xmyTe9GYg1l9a69psu" crossorigin="anonymous"&gt;
        &lt;link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap-theme.min.css" integrity="sha384-6pzBo3FDv/PJ8r2KRkGHifhEocL+1X2rVCTTkUfGk7/0pbek5mMa1upzvWbrUbOZ" crossorigin="anonymous"&gt;
        &lt;script src="https://stackpath.bootstrapcdn.com/bootstrap/3.4.1/js/bootstrap.min.js" integrity="sha384-aJ21OjlMXNL5UyIl/XNwTMqvzeRMZH2w8c5cRVpzpU8Y5bApTppSuUkhZXN0VxHd" crossorigin="anonymous"&gt;&lt;/script&gt;
        &lt;style&gt;
            #video-element {
                width: 500px;
                height: 375px;
                background-color: #666;
            }
            #snapshot-button {
                margin-top: 180px;
            }
            #registration-box {
                margin-top: 50px;
            }
            .hidden{
                display: none;
            }
        &lt;/style&gt;
    &lt;/head&gt;
    &lt;body&gt;
        &lt;div&gt;
            &lt;div id="container" class="row"&gt;
                &lt;div class="col-md-5"&gt;
                    &lt;video autoplay="true" id="video-element"&gt;&lt;/video&gt;
                &lt;/div&gt;
                &lt;div class="col-md-1"&gt;
                    &lt;button id="snapshot-button" class="btn btn-primary"&gt;Snapshot!&lt;/button&gt;
                &lt;/div&gt;
                &lt;div id="registration-box" class="col-md-3 hidden"&gt;
                    &lt;canvas id="canvas" width=240 height=180&gt;&lt;/canvas&gt;
                    &lt;p&gt;Register as:&lt;/p&gt;
                    &lt;div class="form-group"&gt;
                        &lt;label for="name"&gt;Name&lt;/label&gt;
                        &lt;input type="name" class="form-control" id="name" placeholder="Name"&gt;
                        &lt;button id="register-button" class="btn btn-success"&gt;Register&lt;/button&gt;
                    &lt;/div&gt;
                &lt;/div&gt;
            &lt;/div&gt;
        &lt;/div&gt;
        &lt;script src="----------- STEP 2.3 HERE -----------"&gt;&lt;/script&gt;
        &lt;script&gt;
            // Initialize the Amazon Cognito credentials provider

            // ----------- STEP 2.4 HERE -----------

            // Initialize Amazon Rekognition object and define face collection Id
            
            // ----------- STEP 2.5 HERE -----------

            // Prepare HTML 5 video and canvas object
            var video = document.querySelector("#video-element");
            const canvas = document.getElementById('canvas');
            const context = canvas.getContext('2d');
            //Initiate webcam playback
            if (navigator.mediaDevices.getUserMedia) {
                navigator.mediaDevices.getUserMedia({ video: true, facingMode: "user" })
                    .then(function (stream) {
                        video.srcObject = stream;
                    })
                    .catch(function (err0r) {
                        console.log("Something went wrong!");
                    });
            }
            // Initialize listener for snapshot action
            $('#snapshot-button').on('click', function () {
                context.drawImage(video, 0, 0, canvas.width, canvas.height);
                $('#registration-box').removeClass("hidden")
            })
            // Initialize listener for face register button
            $('#register-button').on('click', function () {
                $btn = $(this)
                if ($('#name').val() == "") alert("Please fill in the name")
                $btn.button("loading")
                // Process image
                var imageBytes = processImage();
                // Define index new face function
                
                // ----------- STEP 2.6 HERE -----------

                // Use/create new collection and index new face
                fetchCollection(indexFace)
            })
        &lt;/script&gt;
        &lt;script&gt;
            // Define function to check whether collection exists, or create new           
            var fetchCollection = function(indexFaceFunction){
                var params = {};
                rekognition.listCollections(params, function(err, data) {
                    var createNewCollection = function(){
                        var params = {
                            CollectionId: faceCollectionId
                        };
                        rekognition.createCollection(params, function(err, data) {
                            if (err) console.log(err, err.stack); // an error occurred
                            else {
                                indexFaceFunction(faceCollectionId)
                            }
                        });
                    }
                    if (err) console.log(err, err.stack); // an error occurred
                    else {
                        if(data.CollectionIds.length == 0){
                            createNewCollection()
                        }else{
                            if(data.CollectionIds.includes(faceCollectionId)){
                                indexFaceFunction(faceCollectionId)
                            }else{
                                createNewCollection()
                            }
                        }
                        
                    }
                });
            }
            // Initialize function to process snapshotted image to bytes
            var processImage = function(){
                var imageDataString = canvas.toDataURL("image/jpeg");
                imageDataString = atob(imageDataString.split("data:image/jpeg;base64,")[1]);  // ArrayBuffer
                var length = imageDataString.length;
                var imageBytes = new ArrayBuffer(length);
                var ua = new Uint8Array(imageBytes);
                for (var i = 0; i &lt; length; i++) {
                    ua[i] = imageDataString.charCodeAt(i);
                }
                return imageBytes
            }
        &lt;/script&gt;
    &lt;/body&gt;
&lt;/html&gt;
</code></pre>
<p><strong>2.3.</strong> Replace <strong>----------- STEP 2.3 HERE -----------</strong> at line 45 with link <strong><a href="https://sdk.amazonaws.com/js/aws-sdk-2.457.0.min.js">https://sdk.amazonaws.com/js/aws-sdk-2.457.0.min.js</a></strong> so that it looks like one below</p>
<pre><code>&lt;script  src="https://sdk.amazonaws.com/js/aws-sdk-2.457.0.min.js"&gt;&lt;/script&gt;
</code></pre>
<p>This step includes AWS Software Development Kit (SDK) for JavaScript into your application, so that the application can use AWS services.<br>
<strong>2.4.</strong> Replace <strong>// ----------- STEP 2.4 HERE -----------</strong> at line 49 with the Cognito identity code you copied at step 1.14 before. With this, the application can now assume role to get permission to access AWS services.<br>
<strong>2.5.</strong> Replace <strong>// ----------- STEP 2.5 HERE -----------</strong>  with the code below:</p>
<pre><code>   var  rekognition = new  AWS.Rekognition();
   var  faceCollectionId = "mycollection"
</code></pre>
<p>This code initialize <strong>rekognition</strong> object to access AWS Rekognition from your application. It also defines <strong>face collection</strong> to store the face feature (not in raw format, but in extracted feature for the deep learning)<br>
<strong>2.6.</strong> Replace <strong>// ----------- STEP 2.6 HERE -----------</strong>  with the code below:</p>
<pre><code>var indexFace = function (collectionId){
    var params = {
        CollectionId: collectionId, 
        Image: { 
            Bytes: imageBytes
        },
        DetectionAttributes: ["ALL"],
        ExternalImageId: $('#name').val(),
        MaxFaces: 1,
        QualityFilter: "NONE"
    };
    rekognition.indexFaces(params, function(err, data) {
        if (err) console.log(err, err.stack); // an error occurred
        else  {
            alert("Successful..")
            $btn.button("reset")
            $("#registration-box").addClass("hidden");
        }
    });
}
</code></pre>
<p>This code calls Rekognition IndexFaces API to index the face so that it can be searched later.<br>
<strong>2.7.</strong> Open the file using your browser (preferably on new version Google Chrome or Mozilla Firefox) and allow the webcam access. Take snapshot of yourself and people next to you.</p>
<h2 id="step-3--build-the-person-identification-web-application">Step 3 : Build The Person Identification Web Application</h2>
<p>Now that we have had the HTML page for registering people, we will build another one for identifying person in front of camera.</p>
<p><strong>3.1.</strong> On your laptop, create another file named <strong>identify.html</strong>.<br>
<strong>3.2.</strong> Paste the following code into the file:</p>
<pre><code>&lt;!doctype html&gt;
&lt;html lang="en"&gt;
   &lt;head&gt;
       &lt;script src="https://code.jquery.com/jquery-3.4.1.min.js" integrity="sha256-CSXorXvZcTkaix6Yvo6HppcZGetbYMGWSFlBw8HfCJo=" crossorigin="anonymous"&gt;&lt;/script&gt;
       &lt;link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap.min.css" integrity="sha384-HSMxcRTRxnN+Bdg0JdbxYKrThecOKuH5zCYotlSAcp1+c8xmyTe9GYg1l9a69psu" crossorigin="anonymous"&gt;
       &lt;link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap-theme.min.css" integrity="sha384-6pzBo3FDv/PJ8r2KRkGHifhEocL+1X2rVCTTkUfGk7/0pbek5mMa1upzvWbrUbOZ" crossorigin="anonymous"&gt;
       &lt;script src="https://stackpath.bootstrapcdn.com/bootstrap/3.4.1/js/bootstrap.min.js" integrity="sha384-aJ21OjlMXNL5UyIl/XNwTMqvzeRMZH2w8c5cRVpzpU8Y5bApTppSuUkhZXN0VxHd" crossorigin="anonymous"&gt;&lt;/script&gt;
       &lt;style&gt;
           #container {
               margin: 0px auto;
               width: 500px;
               height: 375px;
               border: 10px #333 solid;
           }
           #videoElement {
               width: 500px;
               height: 375px;
               background-color: #666;
           }
       &lt;/style&gt;
   &lt;/head&gt;
   &lt;body&gt;
       &lt;div id="container"&gt;
           &lt;video autoplay="true" id="videoElement"&gt;&lt;/video&gt;
           &lt;p id="identified"&gt;&lt;/p&gt;
           &lt;canvas id="canvas" width=320 height=240 style="display:none"&gt;&lt;/canvas&gt;
       &lt;/div&gt;
       &lt;script src="https://sdk.amazonaws.com/js/aws-sdk-2.457.0.min.js"&gt;&lt;/script&gt;
       &lt;script&gt;
           // Initialize the Amazon Cognito credentials provider
           
           // ----------- STEP 3.3 HERE -----------

           // Initialize Amazon Rekognition object
           var rekognition = new AWS.Rekognition();
           // Prepare HTML 5 video and canvas object
           var video = document.querySelector("#videoElement");
           const canvas = document.getElementById('canvas');
           const context = canvas.getContext('2d');
           // Initiate webcam playback
           if (navigator.mediaDevices.getUserMedia) {
               navigator.mediaDevices.getUserMedia({ video: true, facingMode: "environment" })
                   .then(function (stream) {
                       video.srcObject = stream;
                   })
                   .catch(function (err0r) {
                       console.log("Something went wrong!");
                   });
           }
           // Search face using Rekognition every second
           var interval = setInterval(function(){
               context.drawImage(video, 0, 0, canvas.width, canvas.height);
               var imageBytes = processImage()
               
               // ----------- STEP 3.4 HERE -----------

           },1000)
       &lt;/script&gt;
       &lt;script&gt;
           // Initialize function to process snapshotted image to bytes
           var processImage = function(){
               var imageDataString = canvas.toDataURL("image/jpeg");
               imageDataString = atob(imageDataString.split("data:image/jpeg;base64,")[1]);  // ArrayBuffer
               var length = imageDataString.length;
               var imageBytes = new ArrayBuffer(length);
               var ua = new Uint8Array(imageBytes);
               for (var i = 0; i &lt; length; i++) {
                   ua[i] = imageDataString.charCodeAt(i);
               }
               return imageBytes
           }
           // Initialize function to update the identification result text
           var updateGreetings = function(personName){
               var p = document.getElementById("identified")
               p.innerHTML = '';
               if(personName != null){
                   p.appendChild(document.createTextNode("Welcome, " + personName))
               }
           }
           // Initialize function to detect the mood of the person using Rekognition and log to DynamodB table
           
           // ----------- STEP 4.4 HERE ----------- 

       &lt;/script&gt;
   &lt;/body&gt;
&lt;/html&gt;
</code></pre>
<p><strong>3.3.</strong> Replace <strong>// ----------- STEP 3.3 HERE -----------</strong> at line 35 with your Cognito identity configuration from step 1.14<br>
<strong>3.4.</strong> Replace <strong>// ----------- STEP 3.4 HERE -----------</strong> with the code below</p>
<pre><code>var params = {
  CollectionId: "mycollection", /* required */
  Image: { /* required */
      Bytes: imageBytes
  },
  FaceMatchThreshold: 0.7,
  MaxFaces: 1
};
rekognition.searchFacesByImage(params, function(err, data) {
  if (err) {
      updateGreetings(null)
      console.log(err);
  } else {
      if(data.FaceMatches.length != 0){
          var personName = data.FaceMatches[0].Face.ExternalImageId;
          updateGreetings(personName);
          // ----------- STEP 4.3 HERE -----------
      }else{
          updateGreetings(null)
      }
  }
});
</code></pre>
<p>This code makes <strong>SearchFacesByImage</strong> API call to Rekognition, along with the image snapshot of the webcam, to identify the person. This code is inside a JavaScript continuous interval being executed every 1 second.<br>
<strong>3.5.</strong> Open the HTML file using your browser (preferable new version of Google Chrome or Mozilla Firefox) be in front of the camera. Get people next to you to try too. <strong>Smile</strong> :)</p>
<h2 id="step-4--add-sentiment-analysis">Step 4 : Add Sentiment Analysis</h2>
<p>Up to step 3, our application should be able to identify person that has ben indexed/registered. Now we want to make it capturing people’s sentiments so that we can analyze it later.</p>
<p><strong>4.1.</strong> Create DynamoDB table by going to <a href="https://console.aws.amazon.com/dynamodb/home">https://console.aws.amazon.com/dynamodb/home</a> and click <strong>Create table</strong>.<br>
<strong>4.2.</strong> For “Table name”, fill in <strong>People</strong>. For “Primary key”, fill in “PersonName”. Check <strong>Add sort key</strong>. For the new sort key, fill in <strong>TS</strong> and change dropdown next to it from “String” to <strong>Number</strong><br>
<strong>4.3.</strong> Click <strong>Create</strong><br>
<strong>4.4.</strong> Open again file <strong>identify.html</strong> and locate <strong>// ----------- STEP 4.4 HERE -----------</strong> and replace it with code below:</p>
<pre><code>detectMood(personName, imageBytes) 
</code></pre>
<p><strong>4.5.</strong> Still in the same file, locate <strong>// ----------- STEP 4.5 HERE -----------</strong> and replace it with code below:</p>
<pre><code>var detectMood = function(personName, faceImage){
  var params = {
      Image: {
          Bytes: faceImage
      },
      Attributes: ["ALL"]
  }
  rekognition.detectFaces(params, function(err, data) {
      if (err) {
          console.log(err);
      } else {
          if(data.FaceDetails.length &gt; 0){
              // Initialize DynamoDB object and table name
              var tableName = "People"
              var dynamodb = new AWS.DynamoDB();
              // Collect emotions data
              var faceData = data.FaceDetails[0];
              var params = { TableName: tableName, Item: {"PersonName": { S: personName }, "TS": { N: Date.now().toString()} }}
              for(var i in faceData.Emotions){
                  if(faceData.Emotions[i].Type == "HAPPY" &amp;&amp; faceData.Emotions[i].Confidence &gt; 70) params.Item.happy = {BOOL: true}
                  if(faceData.Emotions[i].Type == "SAD" &amp;&amp; faceData.Emotions[i].Confidence &gt; 7) params.Item.sad = {BOOL: true}
                  if(faceData.Emotions[i].Type == "ANGRY" &amp;&amp; faceData.Emotions[i].Confidence &gt; 10) params.Item.angry = {BOOL: true}
                  if(faceData.Emotions[i].Type == "CONFUSED" &amp;&amp; faceData.Emotions[i].Confidence &gt; 50) params.Item.calm = {BOOL: true}
                  if(faceData.Emotions[i].Type == "CALM" &amp;&amp; faceData.Emotions[i].Confidence &gt; 70) params.Item.calm = {BOOL: true}
              }
              if(faceData.Smile.Value &amp;&amp; faceData.Smile.Confidence &gt; 70)  params.Item.smile = {BOOL: true}
              // Log the data to DynamoDB table
              dynamodb.putItem(params, function(err, data) {
                  if (err) {
                      console.log(err);
                  } 
              });
          }
      }
  });
}
</code></pre>
<p>The two code blocks we added just know will call <strong>DetectFaces</strong> API to Rekognition for the face identified to fetch the emotions data. The emotions data is then streamed to DynamoDB to be stored.<br>
<strong>4.6.</strong> Create new file named <strong>analyze.html</strong> and paste the following code:</p>
<pre><code>&lt;!doctype html&gt;
&lt;html lang="en"&gt;
    &lt;head&gt;
        &lt;script src="https://code.jquery.com/jquery-3.4.1.min.js" integrity="sha256-CSXorXvZcTkaix6Yvo6HppcZGetbYMGWSFlBw8HfCJo=" crossorigin="anonymous"&gt;&lt;/script&gt;
        &lt;link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap.min.css" integrity="sha384-HSMxcRTRxnN+Bdg0JdbxYKrThecOKuH5zCYotlSAcp1+c8xmyTe9GYg1l9a69psu" crossorigin="anonymous"&gt;
        &lt;link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/3.4.1/css/bootstrap-theme.min.css" integrity="sha384-6pzBo3FDv/PJ8r2KRkGHifhEocL+1X2rVCTTkUfGk7/0pbek5mMa1upzvWbrUbOZ" crossorigin="anonymous"&gt;
        &lt;script src="https://stackpath.bootstrapcdn.com/bootstrap/3.4.1/js/bootstrap.min.js" integrity="sha384-aJ21OjlMXNL5UyIl/XNwTMqvzeRMZH2w8c5cRVpzpU8Y5bApTppSuUkhZXN0VxHd" crossorigin="anonymous"&gt;&lt;/script&gt;
        &lt;link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/2.8.0/Chart.min.css"&gt;
        &lt;script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/2.8.0/Chart.bundle.min.js"&gt;&lt;/script&gt;
    &lt;/head&gt;
    &lt;body&gt;
        &lt;div id="container"&gt;
        &lt;/div&gt;
        &lt;script src="https://sdk.amazonaws.com/js/aws-sdk-2.457.0.min.js"&gt;&lt;/script&gt;
        &lt;script&gt;
            // Initialize the Amazon Cognito credentials provider
            AWS.config.region = 'us-east-1'; // Region
            AWS.config.credentials = new AWS.CognitoIdentityCredentials({
                IdentityPoolId: 'us-east-1:13301233-6e61-4a79-9b12-b0d2883c9e2b',
            });
            // Initialize DynamoDB and Rekognition client object and define table name
            var dynamodb = new AWS.DynamoDB();
            var rekognition =  new AWS.Rekognition();
            var tableName = "People"
            // List all persons whose face already indexed
            var params = {
                CollectionId: "mycollection", /* required */
                MaxResults: 100,
            };
            rekognition.listFaces(params, function(err, data) {
                if (err) console.log(err); // an error occurred
                else  {
                    // Grab emtions data from last 30 takes of each person
                    for(var i in data.Faces){
                        const personName = data.Faces[i].ExternalImageId
                        constructDataRow(personName);
                        var params = {
                            ExpressionAttributeValues: {
                            ":v1": {
                                S: personName
                                }
                            }, 
                            KeyConditionExpression: "PersonName = :v1", 
                            TableName: tableName,
                            Limit: 100
                        };  
                        dynamodb.query(params, function(err, data) {
                            if (err) console.log(err); // an error occurred
                            else  {
                                try{
                                    displayBar(data);
                                    displayLine(data);
                                }
                                catch(e){console.log(e)}
                            }           
                        });
                    }
                }
            });
        &lt;/script&gt;
        &lt;script&gt;
            function displayLine(data){
                if(data.Items.length &gt; 0){
                    const personName = data.Items[0].PersonName.S
                    const barChartContainer = $("#" + personName + "-linechart")
                    barChartContainer.contents().remove()
                    barChartContainer.append($("&lt;canvas&gt;&lt;/canvas&gt;").attr({id: personName + "-linecanvas"}));
                    const ctx = document.getElementById(personName + '-linecanvas').getContext('2d');
                    const xLabels = data.Items.map(function(d){
                        const ds = new Date(parseInt(d.TS.N)).toUTCString()
                        const da = ds.split(" ");
                        const x = da[1] + "/" + da[2] + "/" + da[3] + "-" + da[4];
                        return x
                    })
                    var newData = data.Items.map(function(d){
                        const ds = new Date(parseInt(d.TS.N)).toUTCString()
                        const da = ds.split(" ");
                        const x = da[1] + "/" + da[2] + "/" + da[3] + "-" + da[4];
                        const isSmile = typeof d.smile != "undefined" &amp;&amp; d.smile.BOOL
                        const isHappy = typeof d.happy != "undefined" &amp;&amp; d.happy.BOOL
                        const isCalm = typeof d.calm != "undefined" &amp;&amp; d.calm.BOOL
                        const isConfused = typeof d.confused != "undefined" &amp;&amp; d.confused.BOOL
                        const isSad = typeof d.sad != "undefined" &amp;&amp; d.sad.BOOL
                        const isAngry = typeof d.angry != "undefined" &amp;&amp; d.angry.BOOL
                        if(isSmile || isHappy) return {y: "Happy"}
                        if(isCalm) return {y: "Calm/Neutral"}
                        if(isConfused || isSad || isAngry) return { y: "Confused/Angry/Sad"}
                        return {y: "Unknown"}
                    });
                    const borderColor = 'rgba(' + Math.random()*255 +', ' + Math.random()*255 + ', '+ Math.random()*255 +', 0.8)'
                    var lineChartData = {
                        datasets: [{
                            label: personName,
                            fill: false,
                            backgroundColor: borderColor,
                            pointHoverBackgroundColor: 'rgba(' + Math.random()*255 +', ' + Math.random()*255 + ', '+ Math.random()*255 +', 0.8)',
                            pointBorderColor: borderColor,
                            showLine: false,
                            borderWidth: 3,
                            data: newData
                        }]
                    };
                    const myBarChart = new Chart(ctx, {
                        type: 'line',
                        data: lineChartData,
                        options: {
                            responsive: true,
                            legend: {
                                position: 'right',
                            },
                            title: {
                                display: true,
                                text: 'Time Series'
                            },
                            scales: {
                                xAxes: [{
                                    type: 'category',
                                    position: 'bottom',
                                    labels: xLabels
                                }],
                                yAxes: [{
                                    type: 'category',
                                    position: 'left',
                                    labels: ["Happy", "Calm/Neutral", "Confused/Angry/Sad"]
                                }]
                            }
                        }
                    });
                }
            }
            function displayBar(data){
                if(data.Items.length &gt; 0){
                    const personName = data.Items[0].PersonName.S
                    const barChartContainer = $("#" + personName + "-barchart")
                    barChartContainer.contents().remove()
                    barChartContainer.append($("&lt;canvas&gt;&lt;/canvas&gt;").attr({id: personName + "-barcanvas"}));
                    const ctx = document.getElementById(personName + '-barcanvas').getContext('2d');
                    var barChartData = {
                        labels: ['Happy', 'Calm/Neutral', 'Confused/Sad/Angry'],
                        datasets: [{
                            label: personName,
                            backgroundColor: 'rgba(' + Math.random()*255 +', ' + Math.random()*255 + ', '+ Math.random()*255 +', 0.8)',
                            hoverBackgroundColor: 'rgba(' + Math.random()*255 +', ' + Math.random()*255 + ', '+ Math.random()*255 +', 0.8)',
                            borderColor: 'rgba(0, 0, 0, 0.1)',
                            borderWidth: 1,
                            data: aggregateMood(data)
                        }]
                    };
                    const myBarChart = new Chart(ctx, {
                        type: 'bar',
                        data: barChartData,
                        options: {
                            responsive: true,
                            legend: {
                                position: 'right',
                            },
                            title: {
                                display: true,
                                text: 'Mood Today'
                            }
                        }
                    });
                }
            }
            function aggregateMood(data){
                var output = [0,0,0]
                for(var i in data.Items){
                    const d = data.Items[i];
                    const filter = Date.now() - data.Items[i].TS.N &lt;= 86400000
                    if(filter){ // If data still within 1 day
                        const isSmile = typeof d.smile != "undefined" &amp;&amp; d.smile.BOOL
                        const isHappy = typeof d.happy != "undefined" &amp;&amp; d.happy.BOOL
                        const isCalm = typeof d.calm != "undefined" &amp;&amp; d.calm.BOOL
                        const isConfused = typeof d.confused != "undefined" &amp;&amp; d.confused.BOOL
                        const isSad = typeof d.sad != "undefined" &amp;&amp; d.sad.BOOL
                        const isAngry = typeof d.angry != "undefined" &amp;&amp; d.angry.BOOL
                        if(isSmile || isHappy) output[0] += 1
                        if(isCalm) output[1] += 1
                        if(isConfused || isSad || isAngry) output[2] += 1
                    }
                }
                return output
            }
            function constructDataRow(personName){
                const row = $("&lt;div&gt;&lt;/div").addClass("row")
                // Attach to container
                $("#container").append(row)
                // Append person name column
                row.append($("&lt;div style='padding-left: 50px;padding-top:80px;'&gt;&lt;/div&gt;").addClass("col-md-1").append(document.createTextNode(personName)))
                // Append bar chart column
                row.append($("&lt;div&gt;&lt;/div&gt;").addClass("col-md-4").css("padding-top","80px").append(document.createTextNode('No Data')).attr({id: personName + "-barchart"}))
                // Append time series column
                row.append($("&lt;div&gt;&lt;/div&gt;").addClass("col-md-7").css("padding-top","80px").append(document.createTextNode('No Data')).attr({id: personName + "-linechart"}))
            }
        &lt;/script&gt;
    &lt;/body&gt;
&lt;/html&gt;
</code></pre>
<p><strong>4.7.</strong> Open the file using your browser (new version of Google Chrome or Mozilla Firefox is preferred), and examine the analysis result.</p>
<p>With this we have finished Lab 1. Of course, you can keep on improving and <strong>innovating</strong> :)</p>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>

