# Integrating Qualtrics and Creyos (aka CBS) Research / Health

## Purpose

This document will give you detailed instructions about how to integrate Qualtrics and Creyos Research. This will allow you to automatically redirect sutdy partcicipants from a Qualtrics survey (e.g., Letter of Information / Consent Form, a demographic questionnaire, email registration page, etc.) to your Creyos trial. Importantly, the redirection will:
- automatically create a Creyos account for the participant (using a subject identifier created by, or passed along by, your Qualtrics survey);
- log them in to your research trial; and
- start the specified batch (i.e., collection of cognitive tasks).

The specifics depend on whether your Creyos data collection is implemented using Creyos' Research or Health platform. Research is standard platform that has been used for many years, whereas Health is a newer platform with a focus on Healthcare service providers; the latter can be (and is) used for running research trials, and will be the only service in the near future. For now, this document descibes integration with Creyos Health.

## How it works
Simply, at the end of your Qualtrics survey each participant will be redirect to a custom Creyos URL, looks like this:

```
https://health.creyos.com/en/account/signup/owen_lab_example_trial?p=eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2NvZGUiOiJjb25vcndpbGQrMTIzNDU2Nzg5MDk4NzEyMzQ2NzYyMzQ3NjIzNDg5NyIsImJpcnRoX3llYXIiOiIxOTI5IiwiZ2VuZGVyIjoiTWFsZSIsImFmdGVyX2F1dGhfcmVkaXJlY3RfcGF0aCI6Imh0dHBzOi8vaGVhbHRoLmNyZXlvcy5jb20vZW4vaGVhbHRoL3Byb3RvY29scy8xMjY1MjMiLCJlcnJvcl91cmwiOiJodHRwOi8vd3d3Lmdvb2dsZS5jb20iLCJleHAiOjE2ODIxNjkzMDl9._IDsGwamLCjQhty_gDGHhJ9X11b5NPQkQWX27iTrD8ChxIatdAu5MDz4LrGzSjC6j5aYCjrRGbQswOTsCAaIow
```

Although it looks pretty random, this URL is made up of two parts: 
1. The Creyos trial signup URL (`https://health.creyos.com/en/account/signup/owen_lab_example_trial`)
2. A query parameter(named p; `?p=`) containing a [JSON Web Token (JWT)](https://jwt.io/). This token encapsulates data about the participant (e.g., ID, batch to be played, etc.) and an encrypted signature that validates the data structure. The trick is automatically create the JWT for each participant!

Finally, we can add another redirection to the end the Creyos batch that will take your participants back to another Qualtrics survey, or somewhere else, and pass along data like the user identifier.

## Instructions
1. Add the following JavaScript snippet to the header of your Qualtrics survey:
```
<script src="https://cdnjs.cloudflare.com/ajax/libs/jsrsasign/10.7.0/jsrsasign-all-min.js" integrity="sha512-At6mAU6yhy8gg1PeLBZkS563zrOrMYyICnWYaymxN/GOHWjKUeBYZ0ubrIjPI/GoaMevqk37mZz+dl6eQQG2WA==" crossorigin="anonymous" referrerpolicy="no-referrer"></script>
```
This code imports an external JS library ([jsrsasign](https://github.com/kjur/jsrsasign)) into your survey webpage, and we're going to use this library to generate JWTs. The following image shows you where to paste the code in Qualtrics.

<center><img src="./images/01-qualtrics.png" alt="Q1" width="200"/></center>

2. Add (at least two) two embedded data fields to your survey: `CREYOS_URL` will hold the redirection URL for a participant, and `SUBJ_ID` will contain their unique study identifier. In Qualtrics' "Survey Flow", add a new element ("Embedded Data") and move it to be the 1st element in the survey flow. Create the two fields. It should look something like this:
<center><img src="./images/02-qualtrics.png" alt="Q2" width="500"/></center>

3. Find the required info about your Creyos Trial. You'll need: 1) the trial `NAME` and 2) the trial `SALT` (a unique hexadecimal ID). Login to [Creyos](https://health.creyos.com/), then paste this in your browser's URL bar: `https://health.creyos.com/en/admin/trials`. Find and select "View" for your trial, then locate `NAME` and `SALT`. Copy and paste these values somewhere safe. *Note that you have to replace spaces in your trial's name with underscores.*

4. Find the batch identifier for the cognitive assessment that you want participants to complete. Go to `https://health.creyos.com/en/admin/batches`, locate the specific batch you want to use, then click "View". Copy/paste the `ID` number (at the very top of the batch page) somewhere safe.

5. Back in Qualtrics add some JavaScript to the very first question / item in your survey. Click on the question, then under "Question Behaviour" choose "JavaScript" (see green arrows below). Alternatively, there may already be a JavaScript button *on* your question (see yellow arrow):
<center><img src="./images/03-qualtrics.png" alt="Q2" width="600"/></center>

  * Replace the `addOnLoad` block with the following code block. Make sure to replace required fields: `your_trial_name`, `trial_salt`, and `your_batch_id` with the values you collected in steps 3-4.


```{JavaScript}
Qualtrics.SurveyEngine.addOnload(function()
{
	const trial_url = 'https://health.creyos.com/en/account/signup/your_trial_name'
	const trial_salt = '4d30f624f06dc14c2b26408787d3aa34d19a4db4a66c08113ffecee8150522f5'
	
	// This is the JWT Header, that describes the type
	var header = {
		alg: 'HS512',
		typ: 'JWT'
	};
	
	var tNow = KJUR.jws.IntDate.get('now');
	var tEnd = KJUR.jws.IntDate.get('now + 1day');   // This will expire the token one day after it is generated.
	
    // This is the JWT data structure, that has info about your subject and trial.
	var payload = {
		user_code: Qualtrics.SurveyEngine.getEmbeddedData('SUBJ_ID'),
		batch_id: your_batch_id, 
  		error_url: "http://www.google.com",
  		exp: tEnd,
	};
	
	
    // Create and sign (encrypt) the JWT data
	var sJWT = KJUR.jws.JWS.sign("HS512", JSON.stringify(header), JSON.stringify(payload), {rstr: trial_salt});

    // Combine the trial's autoregister URL with the JWT as a query parameter
	var creyos_autoregister_url = trial_url+"?p="+sJWT
	
    // Set the embedded data variable
	Qualtrics.SurveyEngine.setEmbeddedData('CREYOS_URL', creyos_autoregister_url)
});
```

 - Note that the `user_code` field of the JWT, which becomes the user identifier in the Creyos dataset, comes from the Qualtrics Embedded Data field `SUBJ_ID`. We created this data field in Step 2 but we haven't populated it yet! That's because there are many ways to do this, depending on _how_ you are generating the subject identifiers. That's beyond the scope of this document (for now), but possibilities include:
    - It is passed as a query parameter from another survey or service that redirects to this survey.
    - It is dynamically created by this survey using JavaScript in this code block (but before the JWT creation).
    - It is set in the contact list information for this survey.
    - Others?

6. Finally, make the survey redict participants to the `CREYOS_URL` contained in the embedded data field by customizing the "End of Survey" block in the Survey flow:

<center><img src="./images/04-qualtrics.png" alt="Q2" width="600"/></center>

7. You can also set your Creyos batch to redirect upon completion by setting the `Redirect URL` field and including the appropriate parameters, like `user_id`.



