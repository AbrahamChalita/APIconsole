<script lang="ts">
    import { Button, Badge, Input, Label, Table, TableBody, TableBodyCell, TableBodyRow, TableHead, TableHeadCell, Spinner, Tabs, TabItem, Alert } from 'flowbite-svelte';

    type Model = {
        model: string;
        class_label: string;
        not_spam_probability: number;
        spam_probability: number;
        predicted_class: number;
    };

    let answerExample = {
        models: [] as Model[]
    };
    
    let loading = false;

    let emailFrom = "";
    let emailSubject = "";
    let attachmentCount = "";
    let attachmentExtension = "";

    const predictSingle = async () => {
        loading = true;
        try {
            const response = await fetch('https://a8c5-172-205-170-63.ngrok-free.app/api/v1/predict', {
                method: 'POST',
                mode: 'no-cors',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({
                    "Email From": emailFrom,
                    "Email Subject": emailSubject,
                    "Attachment Count": attachmentCount,
                    "Attachment Extension": attachmentExtension
                })
            });

            if (!response.ok) {
                const errorData = await response.json();
                throw new Error(`HTTP error! status: ${response.status}, message: ${errorData.error}`);
            }

            const data = await response.json();
            answerExample = data;
        } catch (error) {
            console.error('Error in predictSingle:', error);
            errorMessage = 'An error occurred while making the prediction.';
            showAlert = true;
        } finally {
            loading = false;
        }
    }

    let activeTab = "Single";

    let file: File;

    let isFileUploaded = false;
    let isCSVFile = false;

    function handleFileUpload(event: Event & { currentTarget: EventTarget & HTMLInputElement; }) {
        const inputElement = event.currentTarget;
        if (inputElement.files) {
            const uploadedFile = inputElement.files[0];
            if (uploadedFile.type === "text/csv") {
                file = uploadedFile;
                isFileUploaded = true;
                isCSVFile = true;
                console.log("CSV file uploaded: ", file.name);
            } else {
                console.log("Uploaded file is not a CSV file: ", uploadedFile.name);
                inputElement.value = '';
                isFileUploaded = false;
                isCSVFile = false;
            }
        }
    }

    function openGithub() {
        window.open("https://github.com/AbrahamChalita/APIconsole", "_blank");
    }

    type PredictionData = {
    id: number;
    models: {
        predicted_class: number;
        class_label: string;
        spam_probability: number;
        model: string;
        not_spam_probability: number;
    }[];
};

async function updateCSV(data: PredictionData[]): Promise<string> {
    const csvData = await file.text();
    const lines = csvData.split('\n');
    const header = lines[0];
    const updatedHeader = `${header};bert_label;bert_probability;svm_label;svm_probability;rnn_label;rnn_probability`;
    const updatedLines = [updatedHeader];

    for (let i = 1; i < lines.length; i++) {
        const line = lines[i];
        const predictionData = data.find(item => item.id === i - 1);

        if (predictionData) {
            const bertPrediction = predictionData.models.find(model => model.model === 'BERT');
            const svmPrediction = predictionData.models.find(model => model.model === 'SVM');
            const rnnPrediction = predictionData.models.find(model => model.model === 'LSTM');

            const bertLabel = bertPrediction?.class_label ?? '';
            const bertProbability = bertPrediction?.spam_probability ?? 0;
            const svmLabel = svmPrediction?.class_label ?? '';
            const svmProbability = svmPrediction?.spam_probability ?? 0;
            const rnnLabel = rnnPrediction?.class_label ?? '';
            const rnnProbability = rnnPrediction?.spam_probability ?? 0;

            const updatedLine = `${line};${bertLabel};${bertProbability};${svmLabel};${svmProbability};${rnnLabel};${rnnProbability}`;
            updatedLines.push(updatedLine);
        } else {
            updatedLines.push(line);
        }
    }

    return updatedLines.join('\n');
}

const predictBatch = async () => {
    try {
        loading = true;
        const formData = new FormData();
        formData.append('file', file);

        const response = await fetch('https://a8c5-172-205-170-63.ngrok-free.app/api/v1/predict_batch', {
            method: 'POST',
            body: formData
        });

        if (!response.ok) {
            const errorData = await response.json();
            throw new Error(`HTTP error! status: ${response.status}, message: ${errorData.error}`);
        }

        const data: PredictionData[] = await response.json();

        const updatedCSV = await updateCSV(data);
        downloadCSV(updatedCSV);

        loading = false;
    } catch (error) {
        console.error('Error in predictBatch:', error);
        errorMessage = 'An error occurred while making the batch prediction.';
        showAlert = true;
        loading = false;
    }
}

function downloadCSV(csvData : string) {
        const blob = new Blob([csvData], { type: 'text/csv;charset=utf-8;' });
        const link = document.createElement('a');
        const url = URL.createObjectURL(blob);
        link.setAttribute('href', url);
        link.setAttribute('download', 'updated_predictions.csv');
        link.style.visibility = 'hidden';
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
    }

    let showAlert = false;
    let errorMessage = '';
    
   
</script>

<div class="main-container">
    {#if showAlert}
        <Alert color="red" dismissable on:dismiss={() => showAlert = false}>
            {errorMessage}
        </Alert>
    {/if}
    <div class="header">
        <div class="title">
            <h1 class="font-bold"> Team 1 </h1>
        </div>
        <div class="doc-item">
            <a href="/documentation" class="hover:underline">Documentation</a>
        </div>
        <div class="github">
            <Button color="alternative" class="hover:text-blue-800"
                on:click={openGithub}>Github</Button>
        </div>
    </div>
    <div class="content">
        <div class="left">
            <h1> Email <span class="text-primary-800">Spam Prediction</span> API </h1>
            <p> API for predicting whether an email is spam or not using a combination of three models: </p>
            <div class="models">
                <Badge large color="primary" class="m-3">SVM</Badge>
                <Badge large color="primary" class="m-3">RNN</Badge>
                <Badge large color="primary" class="m-3">BERT</Badge>
            </div>
            <p> API used for experimenting and testing the models. </p>
        </div>
        <div class="right">
            <div class="input-square">
                <div class="input-title">
                    <Tabs>
                        <TabItem title="Single" open
                            on:click={
                                () => {
                                    activeTab = "Single";
                                }
                            }
                        ></TabItem>
                        <TabItem title="Batch" on:click={
                            () => {
                                activeTab = "Batch";
                            }
                        }></TabItem>
                    </Tabs>
                    <span class="mr-10"> Try it </span>
                </div>
                {#if activeTab === "Single"}
                <div class="entries">
                    <div class="mb-6">
                        <label for="emailFrom" class="block mb-2">Email From: </label>
                        <Input id="emailFrom" size="lg" bind:value={emailFrom} placeholder="E.G. John Doe <john.doe@example.com>" />
                    </div>
                    <div class="mb-6">
                        <label for="emailSubject" class="block mb-2">Email Subject: </label>
                        <Input id="emailSubject" size="lg" bind:value={emailSubject} placeholder="E.G. Important: Project Update - Q3 Results" />
                    </div>
                    <div class="mb-6">
                        <label for="attachmentCount" class="block mb-2">Attachment Count: </label>
                        <Input id="attachmentCount" size="lg" bind:value={attachmentCount} placeholder="E.G. 3" />
                    </div> 
                    <div class="mb-6">
                        <label for="attachmentExtension" class="block mb-2">Attachment Extension: </label>
                        <Input id="attachmentExtension" size="lg" bind:value={attachmentExtension} placeholder="pdf, pdf, xlsx" />
                    </div>
                </div>
                <div class="flex items-center justify-start ml-5 mb-10">
                    <Badge color="blue" class="m-3">POST</Badge>
                    <spam> localhost:5000/v1/api/predict </spam>
                </div>
                <div class="flex justify-center items-center flex-col">
                    <Button class="w-1/2 ml-10 bg-primary-700"
                        on:click={predictSingle}>
                        {#if loading}
                            <Spinner color="white" size={7}/>
                        {:else}
                            Predict
                        {/if}
                    </Button>
                </div>
                <div class="result-container">
                    {#if Object.keys(answerExample).length > 0 && answerExample.models.length > 0}
                        <Table shadow>
                            <TableHead>
                                <TableHeadCell>Model</TableHeadCell>
                                <TableHeadCell>Label</TableHeadCell>
                                <TableHeadCell>Probability</TableHeadCell>
                            </TableHead>
                            <TableBody>
                                {#each answerExample.models as model}
                                    <TableBodyRow>
                                        <TableBodyCell>{model.model}</TableBodyCell>
                                        <TableBodyCell>
                                        {#if model.class_label == "Spam"}
                                                <Badge color="red">SPAM</Badge>
                                            {:else}
                                                <Badge color="green">Not SPAM</Badge>
                                            {/if}
                                        </TableBodyCell>
                                        <TableBodyCell>
                                            {(model.class_label == "Spam" ? model.spam_probability : model.not_spam_probability).toFixed(8)}
                                        </TableBodyCell>
                                    </TableBodyRow>
                                {/each}
                            </TableBody>
                        </Table>
                    {/if}
                </div>
                {:else}
                <div class="batch-container">
                    <input type="file" on:change={handleFileUpload} accept=".csv" />
                    {#if !isCSVFile && isFileUploaded}
                        <Badge color="red"> Not a CSV file </Badge>
                    {/if}
                    {#if isFileUploaded && isCSVFile}
                        <Badge color="green"> Uploaded </Badge>
                    {/if}
                    {#if isFileUploaded && isCSVFile}
                        <div class="flex justify-center items-center flex-col">
                            <Button class="w-1/2 ml-10 mt-10 bg-primary-700"
                                on:click={predictBatch}>
                                {#if loading}
                                    <Spinner color="white" size={7}/>
                                {:else}
                                    Predict
                                {/if}
                            </Button>
                        </div>
                    {/if}
                </div>
            {/if}
            </div>
        </div>
    </div>

</div>


<style>
    .main-container {
        display: flex;
        flex-direction: column;
        height: 100vh;
    }

    .header {
        background-color: #ffffff;
        height: 5%;
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 0 5rem;
    }

    .content {
        background-color: #f4fbff;
        display: flex;
        height: 95%;
    }

    .left {
        width: 50%;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
    }

    .left h1 {
        font-size: 2.8rem;
        font-weight: 700;
    }

    .left p {
        font-size: 1.3rem;
        font-weight: 400;
        color: #6b7280;
        padding-left: 6rem;
        padding-right: 6rem;
        padding-top: 2rem;
    }
    

    .right {
        width: 50%;
        background-color: #ffffff;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
    }

    .input-square {
        width: 80%;
        height: auto;
        background-color: #f4fbff;
        border-radius: 10px;
        box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
    }

    .input-title {
        font-size: 1.2rem;
        font-weight: 700;
        padding-top: 2rem;
        padding-bottom: 2rem;
        padding-left: 2rem;
        text-align: left;
        display: flex;
        justify-content: space-between;
        width: 100%;
    }

    .entries {
        padding-left: 2rem;
        padding-right: 2rem;
    }

    .result-container {
        margin-top: 2rem;
        justify-content: center;
        align-items: center;
        margin-left: 4rem;
        margin-right: 4rem;
        padding-bottom: 2rem;
    }

    .batch-container {
        padding-left: 2rem;
        padding-right: 2rem;
        padding-bottom: 2rem;
    }

    @media (max-width: 768px) {
        .content {
            flex-direction: column;
        }
        .left, .right {
            width: 100%;
        }

        .input-square {
            width: 90%;
            margin-top: 2rem;
        }
    }


    @media (max-width: 768px) {
    .content {
        grid-template-columns: 1fr;
    }
    } 
</style>