# Multi Sequence Learning Experiment-Predict Next Element Azure Cloud Implementation
(Software Engineering Project: ML21/22 Analyze and describe boosting algorithm)

## Project Description
This project describes the functioning of Multi sequence learning for the given sequence of Numbers which includes learning of a sequence of numbers and resulting in the prediction of next element in the sequence based on the learning in HTM Prediction Engine and HPA controller for stability. This Multi sequence learning is Migrated to Azure Cloud for better functioning and user experience.

Multi sequence learning algorithm consists two parts:
1. Learning Phase
2. Prediction Phase

### Learning Phase:
Learning Phase involves training of the given input Sequence of Numbers using HTM and HPA controller for several iterations. After several iterations it attains stability and the spatial pooler enters a stable state.

[Code to Learning Phase](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/2a2ab93ec6bbb0107d8619ea2cf8e44f98edaf82/Source/MyCloudProjectSample/MyExperiment/MultiSequenceLearning.cs)

### Prediction Phase:

After the learning/training phase, the training model generates the similarity matrix for all the classes. The SDRs computed for numbers compare with the SDRs of the respective Sequence learned during the training phase and compute accuracy based on total number of matches and sequence count. Further, the input sequence is computed as SDR input and compared with each of the SDRs of the Sequence learned during the training phase; the best match is searched in the correlation matrix and based on the accuracy and observation class (Label), the Sequence is predicted.

PredictNextElement:

```csharp
private void PredictNextElement(Predictor predictor, double[] list)
        {

            Trace.WriteLine("------------------------------");
            foreach (var item in list)
            {
                var res = predictor.Predict(item);

                if (res.Count > 0)
                {
                    foreach (var pred in res)
                    {
                        Trace.WriteLine($"{pred.PredictedInput} - {pred.Similarity}");
                    }
                    var tokens = res.First().PredictedInput.Split('_');
                    var tokens2 = res.First().PredictedInput.Split('-');
                    PredictedSequence = tokens[0];
                    PredictedNextElement = tokens2.Last();
                    Trace.WriteLine($"Predicted Sequence: {tokens[0]}, predicted next element {tokens2.Last()}");
                }
                else
                {
                    PredictedSequence = "Nothing predicted";
                    PredictedNextElement = "Nothing predicted";
                    Trace.WriteLine("Nothing predicted :(");
                }
            }

            Trace.WriteLine("------------------------------");
        }
```

## 2. What is the input?

Mutiple sequence of numbers are uploaded as a file in 'training-files' blob container as input which is extracted by the Multi sequence learning experiment from the blob while executing. 

We also provide multiple number of sequences from the queue message as input to predixt the next element.

The input file is downloaded from the blob container by using DownloadInputFile method.

DownloadInputFile:

```csharp
public async Task<string> DownloadInputFile(string fileName)
        {
            BlobContainerClient container = new BlobContainerClient(this.config.StorageConnectionString, this.config.TrainingContainer);
            await container.CreateIfNotExistsAsync();
            try
            {
                string BasePath = AppDomain.CurrentDomain.BaseDirectory;
                string localFilePath = Path.Combine(BasePath, fileName);
                string downloadFilePath = localFilePath.Replace(".txt", "DOWNLOADED.txt");
                Trace.WriteLine($"\nDownloading blob to\n\t{downloadFilePath}\n");
                // Get a reference to a blob named "sample-file"
                BlobClient blob = container.GetBlobClient(fileName);
                if (await blob.ExistsAsync())
                {
                    // Download the blob's contents and save it to a file
                    await blob.DownloadToAsync(downloadFilePath);
                    return downloadFilePath;
                }
                else
                {
                    throw new FileNotFoundException();
                }
                
            }
            catch (Exception ex)
            {
                throw;
            }
        }
```

  [Link to sample input file 'SequencesData.txt'](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/f69834f2d767c5614e4fda3ba1a2357a8ea8c5be/Source/MyCloudProjectSample/Documentation/nessesary%20images/Files%20needed%20for%20documentation/SequencesData.txt)

 
The sequences from input sequence file and the input queue message for prediction sequence are parsed using ProcessSequenceData method and ProcessPredictionData method.

ProcessSequenceData:

```csharp
public Dictionary<string, List<double>> ProcessSequenceData(string DataFile)
        {
            Dictionary<string, List<double>> sequences = new Dictionary<string, List<double>>();
            string text = File.ReadAllText(DataFile);
            string[] lines = File.ReadAllLines(DataFile);
            int sequencecount = 1;
            foreach (string line in lines)
            {
                List<double> sequencelist = new List<double>();
                string[] listvalue = line.Split(',');
                foreach (var sub in listvalue)
                {
                    double convertedvalue;
                    double.TryParse(sub, out convertedvalue);
                    sequencelist.Add(convertedvalue);
                }
                string sequencestring = "S" + sequencecount.ToString();
                sequences.Add(sequencestring, sequencelist);
                sequencecount++;
            }
            return sequences;
        }
```

ProcessPredictionData:

```csharp
public double[] ProcessPredictionData(string PredictionSequence)
        {
            string[] listvalue = PredictionSequence.Split(',');
            double[] sequencelist = new double[listvalue.Length];
            for (int i = 0; i < listvalue.Length; i++)
            {
                var convertedvalue = Convert.ToDouble(listvalue[i]);
                sequencelist[i] = convertedvalue;
            }

            return sequencelist;
        }
```
## What is the Output?

After we give the trigger message from the queue for the experiment to download the input sequence file from blob and start the learning phase which then generates the results based on the sequences in the input file. Then we send the queue message again with a prediction sequence to predict the next element.

Two methods UploadExperimentResult and UploadPredictionResult were implemented for uploading experiment results and prediction results into azure table storage. Another method UploadResultFile is implemented to upload output results file into azure blob container.

## How to run experiment

### Connection string

The below mentioned connection string is used for CRUD operations on azure blob container, azure table storage and azure queue. With the help of this connection string, we can upload input sequence file and input queue messages which are required to start the execution of the experiment.

~~~
DefaultEndpointsProtocol=https;AccountName=2022cloudproject;AccountKey=cCAaacBlvk7ftPp3Bqx2Lg/qSy4NZ5pWI9XpFpJNtOJIx3XQkTQpm4Yigzaq4ldsa8lGVTb9qvwl+AStli5wOA==;EndpointSuffix=core.windows.net
~~~

![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/866a313c69d11974944e06e4f4cfaecd2580aa96/Source/MyCloudProjectSample/Documentation/nessesary%20images/Azure-Connection-String.png)

### Step 1. Create training-files container and upload input sequence file

Firstly we need to create a blob container with name **training-files** and then upload the input sequence file into that container.

![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/866a313c69d11974944e06e4f4cfaecd2580aa96/Source/MyCloudProjectSample/Documentation/nessesary%20images/sequencedataimage.png)

you can download the sample input file from the below mentioned link 

 [SequencesData.txt](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/f69834f2d767c5614e4fda3ba1a2357a8ea8c5be/Source/MyCloudProjectSample/Documentation/nessesary%20images/Files%20needed%20for%20documentation/SequencesData.txt)

We can create the container and upload the input sequence file using az cli and connection string of azure storage account

~~~
az storage container create --account-name 2022cloudproject --name training-files --connection-string "DefaultEndpointsProtocol=https;AccountName=2022cloudproject;AccountKey=cCAaacBlvk7ftPp3Bqx2Lg/qSy4NZ5pWI9XpFpJNtOJIx3XQkTQpm4Yigzaq4ldsa8lGVTb9qvwl+AStli5wOA==;EndpointSuffix=core.windows.net"
~~~

![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/866a313c69d11974944e06e4f4cfaecd2580aa96/Source/MyCloudProjectSample/Documentation/nessesary%20images/container%20create.png)

~~~
az storage blob upload --container-name training-files -f "/Users/sandhyabagadi/Downloads/SequencesData.txt" --name SequencesData.txt --connection-string "DefaultEndpointsProtocol=https;AccountName=2022cloudproject;AccountKey=cCAaacBlvk7ftPp3Bqx2Lg/qSy4NZ5pWI9XpFpJNtOJIx3XQkTQpm4Yigzaq4ldsa8lGVTb9qvwl+AStli5wOA==;EndpointSuffix=core.windows.net"
~~~

![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/866a313c69d11974944e06e4f4cfaecd2580aa96/Source/MyCloudProjectSample/Documentation/nessesary%20images/Blob%20upload.png)

Note: 
1. The file path mentioned above should be replaced with local file path where the input sample resides in the pc.
2. Sequence length must contains atleast 3 and atmost 20. Limitation of the multisequence learning experiment.

### Step 2. Pulling docker image from azure container registry

we can pull the experiment docker image from azure container registry as given below

~~~
docker login -u 2022cloudproject -p B7uXzjtBiIPfbZ/DhjQNRYVharyctBIO 2022cloudproject.azurecr.io
~~~
~~~
docker pull 2022cloudproject.azurecr.io/dockerimage1:v1
~~~
![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/866a313c69d11974944e06e4f4cfaecd2580aa96/Source/MyCloudProjectSample/Documentation/nessesary%20images/Docker%20pull.png)

### Step 3. Run the docker image in azure container instance

we can run the multisequence learning experiment docker image in azure container instance with the help of az cli.

registry login server, registry username and registry-password  which are required to create the azure container instance are given below

registry-login-server:
~~~
2022cloudproject.azurecr.io
~~~

registry-username:
~~~
2022cloudproject
~~~

registry-password:
~~~
B7uXzjtBiIPfbZ/DhjQNRYVharyctBIO
~~~

Creating resource group
~~~
az group create --location "East US" -name 2022cloudcomputingproject
~~~

Creating Azure container instance
~~~
az container create --resource-group cloudprojectsandhya --name ccproject1  --image 2022cloudproject.azurecr.io/dockerccproject:v2 --registry-login-server 2022cloudproject.azurecr.io --registry-username 2022cloudproject --registry-password B7uXzjtBiIPfbZ/DhjQNRYVharyctBIO --ports 80
~~~
![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/acbf3914f72b3f6ba67df20832d54cec38ef62cc/Source/MyCloudProjectSample/Documentation/nessesary%20images/Creating%20a%20container%20instance.png)

### Step 4. Download Azure storage explorer

Download Azure storage explorer with this ([Download Link](https://azure.microsoft.com/en-us/products/storage/storage-explorer/)) to send queue messages and view result tables.

configure the storage explorer as shown below

![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/acbf3914f72b3f6ba67df20832d54cec38ef62cc/Source/MyCloudProjectSample/Documentation/nessesary%20images/storage_browser(explorer).png)

Enter the following connection string
~~~
DefaultEndpointsProtocol=https;AccountName=2022cloudproject;AccountKey=cCAaacBlvk7ftPp3Bqx2Lg/qSy4NZ5pWI9XpFpJNtOJIx3XQkTQpm4Yigzaq4ldsa8lGVTb9qvwl+AStli5wOA==;EndpointSuffix=core.windows.net
~~~

### Step 5. Queue Message that will trigger the experiment:

Add a queue message in "trigger-queue" in storage browser with the help of storage account added in step 4. **The encoding must be change to "UTF8"**.

![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/b173cb039851a206b824c872a38388c27cc722ac/Source/MyCloudProjectSample/Documentation/nessesary%20images/Queue-message-to-trigger.png)


Queue message to start the experiment
~~~json
{
    "Name":"multisequencelearning",
    "Description":"learningphase",
    "ExperimentId":"test2",
    "InputFile":"SequencesData.txt"
}
~~~
1. Name: Name of the experiment.
2. Description: Description of the experiment.
3. ExperimentId : Id of the experiment to run.
4. InputFile: Name of the text file with 1000 number of sequences, used for training the model. Stored in training-files azure container.

Queue message to predict the next element
~~~json
{
    "Name":"multisequencelearning",
    "Description":"predictionphase",
    "ExperimentId": "test2",
    "PredictionSequences": "2.0,6.0,19.0" 
}
~~~
1. Name: Name of the experiment.
2. Description: Description of the experiment.
3. ExperimentId : Id of the experiment to run.
4. PredictionSequences: The sequence which needs to be predicted against the learned sequences

Queue message to end the experiment
~~~json
{
    "PredictionSequences": "none" 
}
~~~

 see the below queue message sample file for more combinations
 
 [Queue Message Samples](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/7047438cbabdbf8ff65b6718492e8625f1a8da29/Source/MyCloudProjectSample/Documentation/nessesary%20images/Files%20needed%20for%20documentation/SampleQueuemessages%20(1).docx)
 
 Note: 
 1. Always send the Experimrnt Learning Queue Message First.
 2. Experiment starting and followed by prediction and finally end messages should be added in the same order and can be added one after another in step 5. 

### Step 6. Results

 Once the experiment is completed the results file and predicted file is uploaded into table section. output file and sequencesdata file uploaded in blob section. Refer the image below.

 ![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/acbf3914f72b3f6ba67df20832d54cec38ef62cc/Source/MyCloudProjectSample/Documentation/nessesary%20images/Results%20table%20inexplorer.png)

 #### Results in cli

![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/acbf3914f72b3f6ba67df20832d54cec38ef62cc/Source/MyCloudProjectSample/Documentation/nessesary%20images/Results%20in%20cli.png)

## Azure Cloud Storage Functionalities used in the project

### Blob container:

There are two blob conatiners used in this project.

1. training-files
2. result-files

![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/acbf3914f72b3f6ba67df20832d54cec38ef62cc/Source/MyCloudProjectSample/Documentation/nessesary%20images/training%20and%20result%20files%20in%20blob.png)

### training-files:

training-files container is used to store the Input sequences file and download this file while running the experiment using DownloadInputFile method.

### result-files:

result-files container is used to store the resultant output file "outputfile.txt" which is uploaded by UploadResultFile method.

UploadResultFile:

```csharp
public async Task UploadResultFile(string filePath)
        {
            string connectionString = this.config.StorageConnectionString;
            string fileName = System.IO.Path.GetFileName(filePath);
            // Get a reference to a container name and then create it
            BlobContainerClient container = new BlobContainerClient(connectionString, this.config.ResultContainer);
            await container.CreateIfNotExistsAsync();
            try
            {
                // Get a reference to a blob
                BlobClient blobClient = container.GetBlobClient(fileName);
                Trace.WriteLine($"Uploading to Blob storage as blob:\n\t {blobClient.Uri.ToString()}\n");
                // Upload data from the local file
                await blobClient.UploadAsync(filePath, true);
                Trace.WriteLine($"Uploaded Result File to {this.config.ResultContainer} container successfully");
            }
            catch (Exception ex)
            {
                throw;
            }
        }
```

Link to sample resultant file is given below

[output.txt](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/acbf3914f72b3f6ba67df20832d54cec38ef62cc/Source/MyCloudProjectSample/Documentation/nessesary%20images/Files%20needed%20for%20documentation/outputfile.txt)
 
### Results stored on Table Storage:
 
#### Training Results:
 1. ExperimentId: Id of the experiment that was ran
 2. Accuracy: The percentage of correct predictions for the test data
 3. Max Match Count: Number of times max accuracy reached in a row
 4. Sequence Pair: Sequence of numbers
 5. Predicted Sequence: The key for the sequences
 6. Elapsed Time: Time elapsed to get 30 max match count for each sequence
 7. Name: Name of the experiment
 8. DurationSec: Total duration of the experiment in seconds
 9. StartTimeUtc: Start time of the experiment
 10. EndTimeUtc: End time of the experiment
 11. Description: Description of the experiment
 
 ![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/acbf3914f72b3f6ba67df20832d54cec38ef62cc/Source/MyCloudProjectSample/Documentation/nessesary%20images/Results%20table%20inexplorer.png)

#### Prediction Results:
 1. ExperimentId: Id of the experiment that was ran.
 2. Name: Name of the experiment.
 3. Description: Description of the experiment.
 4. StartTimeUtc: Start time of the experiment.
 5. EndTimeUtc: End time of the experiment.
 6. DurationMilliSec: Total duration of the experiment in Milli seconds.
 7. sequence: Given input sequences.
 8. PredictionSequence: The predicted sequence in the experiment.
 9. Predictednextelement: The number which is predicted next in the sequence.
 
 ![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/acbf3914f72b3f6ba67df20832d54cec38ef62cc/Source/MyCloudProjectSample/Documentation/nessesary%20images/predicted%20results%20explorer.png)

## Running Docker Image in Azure Container Instance:

### Build, Tag and Push the Docker file

Building a docker image using the Dockerfile found at [a link](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/kirankumar-athirala/Source/MyCloudProjectSample/MyCloudProject/Dockerfile)

use the following commands to do so
~~~
docker build -f ./MyCloudProject/Dockerfile -t dockerccproject:v2 .
~~~

![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/acbf3914f72b3f6ba67df20832d54cec38ef62cc/Source/MyCloudProjectSample/Documentation/nessesary%20images/Building%20the%20docker%20image.png)

~~~
docker tag 2022cloudproject.azurecr.io/dockerccproject:v2
~~~
~~~
docker push 2022cloudproject.azurecr.io/dockerccproject:v2
~~~
![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/acbf3914f72b3f6ba67df20832d54cec38ef62cc/Source/MyCloudProjectSample/Documentation/nessesary%20images/Docker%20tagging%20and%20pushing.png)

#### Docker Images
![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/acbf3914f72b3f6ba67df20832d54cec38ef62cc/Source/MyCloudProjectSample/Documentation/nessesary%20images/Docker%20image.png)

### Azure Container Registry

Login Server: 2022cloudproject.azurecr.io

<details>
  <summary>ACR Credentials</summary>
Username:
2022cloudproject

Password:
B7uXzjtBiIPfbZ/DhjQNRYVharyctBIO
</details>

![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/acbf3914f72b3f6ba67df20832d54cec38ef62cc/Source/MyCloudProjectSample/Documentation/nessesary%20images/Azure%20container%20registry.png)

### Azure Container Instance

The Azure container instance is then created with the previously stored docker image in the container registry

![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/b173cb039851a206b824c872a38388c27cc722ac/Source/MyCloudProjectSample/Documentation/nessesary%20images/container-instance.png)

### Via Docker Desktop 

![](https://github.com/UniversityOfAppliedSciencesFrankfurt/se-cloud-2021-2022/blob/acbf3914f72b3f6ba67df20832d54cec38ef62cc/Source/MyCloudProjectSample/Documentation/nessesary%20images/dockerdesktop.png)
