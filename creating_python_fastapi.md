# Creating a model prediction model API server

This guide walks through how to create a python-based, stateless API for serving model predictions.

Tools:
- OpenAPI for defining the endpoint contract.
- pydantic for defining the data model.
- FastAPI for creating the server.

This will roughly outline the steps I took to create the DirectMailAPI server. But I'll use a different example of a dummy server called MyModelAPI.

# Step 0. Make your virtual env

`python -m venv venv`

I won't give all the installs needed for the project. If a package is missing, install it.

# Step 1. Create the OpenAPI spec

This is always the best place to start because it defines the endpoint---both literally and figuratively. Canonically, the name of the file is api.yml and it lives in the root directory.




    openapi: '3.0.2'
    info:
    title: MyModelAPI
    version: '0.1'
    servers:
    - url: http://localhost:8000/
        description: Local development server.
    paths:
    
    /model_prediction
        description: Returns the prediction
        post:
        requestBody:
            content:
            application/json:
                schema:
                type: object
                properties:
                    instance_data:
                    $ref: '#/components/schemas/Instance_Data'
                    version:
                    type: string
                    enum:
                        - v0.0
        responses:
            '200':
            description: OK
            content:
                application/json:
                schema:
                    type: object
                    properties:
                    instance_data:
                        $ref: '#/components/schemas/Instance_Data'
                    prediction:
                        $ref: '#/components/schemas/Prediction'

The data should be defined in its own model schema. We also want to define the prediction data.

    components:
        schemas:
            Instance_Data:
                type: object
                description: An instance of predictors.
                required: [predictor_1, predictor_2, predictor_3, predictor_4]
                properties:
                    predictor_1:
                        type: number
                        minimum: 0
                    predictor_2:
                        type: number
                        minimum: 0
                        maximum: 100
                    predictor_3:
                        type: string
                        enum:
                            - Gold
                            - Silver
                            - Bronze
                    predictor_4:
                        integer
                        minimum: 1
            Prediction:
                type: object
                required: ['version', 'predict_flag','probability']
                properties:
                    version:
                        type: string
                        enum:
                        - v0.0
                    predict_flag:
                        type: integer
                        minimum: 0
                        maximum: 1
                    probability:
                        type: number
                        minimum: 0
                        maximum: 1

With this, we know what the endpoints need to be structured like. We can use the `datamodel-codegen` package to use this file and generate code. It can be placed in its own directory, but we'll generate it here to start.

    datamodel-codegen --input apy.pml --output data_model.py

This will give us some starter code for the pydantic models.

# Step 2. Refine pydantic models

The code generator will have taken care of most of the work, but it misses some things. So here's some stuff to look out for.

We'll need to add a `model_config` at the beginning of the main class. This will alow the enum fields to render as strings instead of objects.

    class Instance_Data(BaseModel):
        model_config = ConfigDict(use_enum_values=True)    # Necessary to convert back to dict correctly.
        
        predictor_1: Annotated[float, Field(strict=True, ge=0.0)]

Also note that we had to change the `predictor_1` field to use `Annotated[..., Field(..., ge=0.0)] `. Older versions of pydantic use function, but this is the preferred method in newer versions.

The only thing we need to add are the request and response models for the API itself. (These didn't generate for me.)

    class App_Request(BaseModel):
        instance_data: Instance_Data
        version: Version

    class App_Response(BaseModel):
        instance_data: Instance_Data
        prediction: Prediction

# Step 3. The server logic

Now create a file in the root directory called `main.py`. This will contain the server logic. Here's some starter code.

    from fastapi import FastAPI
    from pydantic import BaseModel
    from src.data_model import DM_Prediction_Request, DM_Prediction_Response, Prediction
    from src.serve_model import get_prediction

    app = FastAPI()

We'll define the `/model_prediction` endpoint. The function below the decorator can be named anything. That will become the formal name of the endpoint. It's probably best to keep the same name as the endpoint itself.

    @app.post("/model_prediction/")
    def model_prediction(app_request: App_Request):
        version_string = app_request.version
        instance_data = app_request.instance_data
        predict_flag, probability = get_prediction(instance_data, version_string)
        prediction_dict = {"version":version_string, "predict_flag":predict_flag, "probability":probability}
        prediction = Prediction(**prediction_dict)
        response = App_Response(**{"instance_data":instance_data,"prediction":prediction})
        return response

This code starts with an `App_Request` instance called `app_request`. This already contains the parsed JSON and validated model. **Note: If a JSON cannot validate, an error will be thrown before this.** We pull out the version string and the instance data. We feed those into a function called `get_prediction` that returns a tuple of the prediction flag and the probability. We'll define this function later on. We then put together a `prediction_dict` dictionary that has all the elements we need for a `Prediction` model instance. We validate (instantiate) that instance. Then we validate (instantiate) an `App_Response` instance, which we'll return. FastAPI will take care of parsing that out into a proper JSON format. There may be a cleaner way to do all this. It's a lot of converting things from one thing to another. But hey, it works.

The `get_prediction` function will pull the model corresponding with the specified version and use that along with the instance data to give us a prediction flag and probability. We'll define that in a separate file next.

# Step 4. Create the `get_prediction` function

Make a new file for this. I named the file `sorve_model.py` referring to the predictive model, not the data model. The word "model" gets confusing here, and honestly I could have come up with a better name.

We also need to make a folder for our pickled model. Since we'll likely have multiple models, lets make it its own directory. We'll pickle a model to that directory and name it "v0.0". Now we have `model_store/v0.0`.

Here's the code that will serve that model version:

    import os
    import pickle
    import pandas as pd
    from data_model import Instance_Data, Version

    MODEL_STORE_DIR = "model_store"

    def get_prediction(instance_data : dict | Instance_Data, version_string : str | Version) -> tuple:
        instance_dict = instance_data.model_dump() if isinstance(instance_data, Instance_Data) else instance_data
        v_string = version_string.value if isinstance(version_string, Version) else version_string

        # Load the correct model version.
        with open(os.path.join(MODEL_STORE_DIR,v_string), 'rb') as f:
            model = pickle.load(f)
        # Convert instance to dataframe.
        instance_df = pd.DataFrame(instance_dict, index=[0])
        # Run predictions
        predict_flag = model.predict(instance_df).item()
        probability = model.predict_proba(instance_df).item(1)
        return (predict_flag, probability)

In addition to loading the correct model version, this converts the instance data to a dict, which itself converts into a one-row pandas dataframe that the model can run `predict` and `predict_proba` on.

# Step 5. Unit tests

Every application should include unit tests, and you may want to create these before the steps above to help debug. This is just the standard unit testing framework. Tests live in `tests` directory. It helps to have a `case_instance.json` file containing a case of an input JSON.

I also always struggle with import contexts in the test directory. There are packages that help wiht a lot of this. But in the interest of keeping things lightweight, here's a boilerplate that can be used with the standard `unittest` framework. This will also import the `case_instance.json` as a string, which is useful for testing the data model.

    import sys, os
    sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.realpath(__file__))))

    import unittest
    import json

    from data_model import App_Request, App_Response    # models we're testing.

    with open("tests/case_instance.json", "r") as f:
        case_instance_raw = f.read()

    case_instance_flag = json.loads(case_instance_raw)

The `sys.path.insert(...)` portion appends to the `PYNOPATH`, which allows us to import packages like we would in the root directory.

We read the file as a string to the object `case_instance_raw`. We can then use `App_Request.model_validate_json(case_instance_raw)` to check that the model validates as expected. We should also probably make some intentionally incorrect cases and make sure those fail. After all, validation is one of the main reasons for using pydantic and FastAPI.

# Step 6. Start the server and test it

Now we can spin up the server locally and test it out. FastAPI always makes a `/docs` endpoint, so that's a good thing to check. It would live at http://localhost:8000/docs.

I recommend using Postman to check the API endpoints. You can configure a local environment to test the local server and a production environment to test prod. If you need to quickly CURL the running server, this would do the trick:

    curl --location 'https://localhost:8000/model_prediction/' \
    --header 'Content-Type: application/json' \
    --data '{
        "instance_data": {
            "predictor_1": 1.0,
            "predictor_2": 2.0,
            "predictor_3": "Silver",
            "predictor_4: 1
        },
        "version": "v0.0"
    }'

# Step 7. Docker and deploy

This section comes from this great article: [How to Delpoy FastAPI to Fly.io](https://ahmadrosid.com/blog/deploy-fastapi-flyio).

First we make the `requirements.txt` file. We also need to make sure that `uvicorn` is installed using `pip install uvicorn`. Then `pip freeze > requirements.txt`.

Then we dockerize the app. Here's the dockerfile.

    # https://hub.docker.com/_/python
    FROM python:3.10-slim-bullseye

    ENV PYTHONUNBUFFERED True
    ENV APP_HOME /app
    WORKDIR $APP_HOME
    COPY . ./

    RUN pip install --no-cache-dir -r requirements.txt

    CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]

No need to build it. We can then create and deploy it to fly, which will take care of the builds.

Last, we use the simple fly cli commands to create and launch the app.

    fly login    # I didn't need this part.
    fly launch
    fly open

That last one just opens the app once it's deployed. If you didn't specify an `@app.get("/")` route then it will fail that initial load until you navigate to an endpoint or the docs.

Note that if you need to make a change and deploy that change, the command is `fly deploy`.

Now test the production endpoint using Postman or a curl.

That's it! We're done!