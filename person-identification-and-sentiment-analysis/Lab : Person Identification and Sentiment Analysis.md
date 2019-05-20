---


---

<h1 id="lab-1--person-identification-and-sentiment-analysis">Lab 1 : Person Identification and Sentiment Analysis</h1>
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
<strong>2.1.</strong> Download this HTML page <a href="https://s3.amazonaws.com/aws-machinelearning-workshop/person-identification-and-sentiment-analysis/html/register.html">https://s3.amazonaws.com/aws-machinelearning-workshop/person-identification-and-sentiment-analysis/html/register.html</a> to a directory in your laptop.<br>
<strong>2.2.</strong> Open the file using your favorite editor<br>
<strong>2.3.</strong> Replace <strong>----------- STEP 2.3 HERE -----------</strong> at line 45 with link <strong><a href="https://sdk.amazonaws.com/js/aws-sdk-2.457.0.min.js">https://sdk.amazonaws.com/js/aws-sdk-2.457.0.min.js</a></strong> so that it looks like one below</p>
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
<p><strong>3.1.</strong> Download this HTML file <a href="https://s3.amazonaws.com/aws-machinelearning-workshop/person-identification-and-sentiment-analysis/html/identify.html">https://s3.amazonaws.com/aws-machinelearning-workshop/person-identification-and-sentiment-analysis/html/identify.html</a> to a directory in your laptop<br>
<strong>3.2.</strong> Open the file using your favorite editor<br>
<strong>3.3.</strong> Replace <strong>// ----------- STEP 3.3 HERE -----------</strong> at line 35 with your Cognito identity configuration from step 1.14<br>
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
          // ----------- STEP 4.4 HERE -----------
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
<strong>4.6.</strong> Open the previous file named <strong>identify.html</strong> and once again, be in front of the camera. This time, try several face expression (e.g. smile, calm, angry). Get your neighbour to join to improve quantity of the data.<br>
<strong>4.7.</strong> Download this HTML file <a href="https://s3.amazonaws.com/aws-machinelearning-workshop/person-identification-and-sentiment-analysis/html/analyze.html">https://s3.amazonaws.com/aws-machinelearning-workshop/person-identification-and-sentiment-analysis/html/analyze.html</a><br>
<strong>4.8.</strong> Open the file using your browser (new version of Google Chrome or Mozilla Firefox is preferred), and examine the analysis result.<br>
<strong>4.9.</strong> You can always re-open file <strong>index.html</strong> to index new persons, capture expressions with file <strong>identify.html</strong>, and analyze it with file <strong>analyze.html</strong></p>
<p>With this we have finished Lab 1. Of course, you can keep on improving the code and <strong>innovating</strong> :)</p>
<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>

