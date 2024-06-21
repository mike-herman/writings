This write-up goes over creating a Genie-based Julia web API. This builds off the Create Genie App write-up.

I assume the initial deployment has happened and a CI/CD pipeline is set up, so everything that gets pushed to GitHub gets deployed via actions. This won't be going over those steps.

Instead, this will go over a development process to create a simple, stateless credit decisioning API.

# Creating a simple applicant check
We'll start with a simple endpoint that performs "preliminary checks", which are checks on the applicant-provided information only. These ensure the applicant meets basic qualifying criteria. In our case, we make sure the applicant is at least 18 years or older.

## Creating the endpoint
In the Genie project directory, open `routes.jl`. Add the following route:

    route("/prelimchecks", method = POST) do
    message = jsonpayload()
    # The `message` dict is structured as follows:
    # - application
    #   - applied_at
    # - information
    #   - applicant
    #       - dob
    message |> GFLOApp.ingest_payload |> GFLOApp.run_preliminary_checks |> GFLOApp.render_response |> json

    end

This will accept requests at the "/prelimchecks" endpoint

We'll go over contracts later, but this assumes the incoming request JSON has the following structure:

- application: Data about the application itself.
  - applied_at: A string timestamp when the application was created.
- information: Bundles of information from distinct sources.
  - applicant: Information submitted by the applicant.
    - dob: A string date of birth.


It does the following steps:

1. Ingest the incoming JSON request using `jsonpayload` function, which converts it to a Julia dict saved in the object `message`.
2. A function (which like all the `GFLOApp.` functions, we have yet to define) called `GFLOApp.ingest_payload` accepts and parses the message. The purpose of this function is to convert the incoming dict to the data model abstractions used by our application. The result of the function is a dictionary containing the abstractions.
3. A function called `GFLOApp.run_preliminary_checks` will accept the dictionary of abstractions and run all the preliminary checks. It returns an abstraction used in our application.
4. A function called `GFLOApp.render_response` will convert the resulting abstraction to a Julia dict.
5. The `json` function then converts that dict into a JSON to send as the response.

We have very intentionally split this into an ingestion part, a logic part, and a rendering part. This allows us to separate the logic and abstractions we use within our application and the logic used to format any results into a response.

To call these functions, we need to import these functions from a module called `GFLOApp`. To do this, add at the top `import GFLOServer.GFLOApp as GFLOApp`.

For now, we'll be keeping all the logic in one module, but we can split this into submodules later when needed.


# Start with the basic app logic

We'll create the app logic in the file `/lib/GFLOApp.jl`. Start it with the typical module template. We'll also import some packages that we know we'll need:

    module GFLOApp

    import Dates
    import DayCounts

    # Logic goes here

    end #module

As a side note, I'm using `import` instead of `using` here intentionally. Using `import` requires us to prefix the functions from that package. This helps us identify the dependencies in our code.

Off-the-bat, we know a few functions that we need to create, so let's add those.

    function ingest_payload(request_dict::Dict, args...; kwargs...)::Dict
        # Convert the request dictionary to abstractions
    end

    function run_preliminary_checks(application::AbstractApplication, applicant::AbstractApplicant)::Vector{AbstractCheckResult}
        # Accept an Application and Applicant type and return a vector (array) of check results.
    end

    function render_response(check_result_list::Vector{CheckResult})::Dict
        # Convert an array of check results to a Dict for the JSON repsonse.
    end

The `ingest_payload` function is straightforward---it will accept the dictionary (and any other arbitrary arguments) and returns a dictionary with the abstractions we need. The arbitrary arguments it also accepts are not used. This isn't strictly necessary, but if arbitrary extra arguments are included then the function won't break. That makes it less fragile but could have unintended consequences.

Now let's look at the `run_preliminary_checks` function. We know it will need to accept a set of fields about the application and a set of fields about the applicant. It will run a set of checks then produce the results of _all_ checks run. We're only planning on adding one check right now. But in most application flows, multiple preliminary checks are run. This is almost certainly an option we'll want in the future. So we'll want to return an array (i.e. vector) of _all_ the check results. Given these requirements, we can identify the `Application`, `Applicant`, and `CheckResult` types that will be our first abstractions. Since we know we'll need this but don't have a specific example yet, we use `Abstract*` specifications for the types. That's not strictly necessary, but it can help us keep things flexible down the line.

The last function `render_response` accepts the array of check results and converts them to a Julia dictionary, which will be directly converted to JSON.

## Creating types

Let's make the `Application` struct. An application represents one or more party's attempt to obtain a credit product. It is time-bound (has a start and an eventual end). And it must eventually have a terminal state. Given these definitions, let's make the struct. We'll also make the abstract type that this belongs to, which is typically good practice.

    abstract type AbstractApplication end

    struct Application <: AbstractApplication
        application_id::String
        created_at::Dates.DateTime
        expiry_deadline::Union{Dates.DateTime, Nothing}
        terminal_state::Union{String,Nothing}

        function Application(
            application_id::String, 
            created_at::Union{Dates.DateTime, String},
            expiry_deadline::Union{Dates.Date, Nothing, String},
            terminal_state::Union{Nothing,String})
            
            input_array = []
            push!(input_array, application_id)
            # Handle strings in created_at. Cannot be empty string.
            input_created_at = created_at isa String ? Dates.DateTime(created_at, Dates.ISODateTimeFormat) : created_at
            push!(input_array, input_created_at)
            # Handle string sin expiry_deadline. Empty string to nothing.
            if (expiry_deadline isa Nothing) | (expiry_deadline isa Dates.DateTime)    # Non-string cases.
                input_expiry_deadline = expiry_deadline
            elseif expiry_deadline == ""    # Empty string case.
                input_expiry_deadline = nothing
            else    # ISO-formatted string date case. Other formats will fail.
                input_expiry_deadline = Dates.Date(expiry_deadline, Dates.ISODateFormat)
            end
            push!(input_array, input_expiry_deadline)
            # Handle empty string terminal_state.
            input_terminal_state = terminal_state == "" ? nothing : terminal_state
            push!(input_array, input_terminal_state)

            new(input_array...)    # Splatting the input array into args.
        end
    end

The `expiry_deadline` field can be a `Date` object or `Nothing`. This lets us either take a given deadline or set it ourselves if that should be part of the business logic. The `terminal_state` field can be `String` or `Nothing`. Generally, an in-progress application has an empty terminal state. But if an application fails checks then we'll set that terminal state to `rejected`.

We also define an inner constructor for this type. We will almost certainly want to accept string arguments in instantiation. This will convert those to the corresponding non-string types like dates or `Nothing`.

Last, we define an _outer_ constructor that accepts keyword arguments instead of sequential arguments. We'll probably use this pattern multiple times. It may be worth rolling it up into a macro at some point. But that's beyond the scope of this.

Now we'll make the `Applicant` struct.

    abstract type AbstractApplicant end

    struct Applicant <: AbstractApplicant
        applicant_id::String
        dob::Date
    end

The applicant simply has an ID and a date of birth. Both _must_ be given.

    abstract type AbstractApplicant end

    struct Applicant <: AbstractApplicant
        applicant_id::String
        dob::Dates.Date

        function Applicant(
            applicant_id::String,
            dob::Union{Dates.Date,String})

            input_array = []
            push!(input_array, applicant_id)
            # Handling string input for dob.
            input_dob = dob isa String ? Dates.Date(dob,Dates.ISODateFormat) : dob
            push!(input_array, input_dob)

            new(input_array...)
        end
    end

    function Applicant(;kwargs...)
        input_array = [
            kwargs[:applicant_id],
            kwargs[:dob]
        ]
        return Applicant(input_array...)
    end

We use the same constructor patters here.

The last type we need to define is the `CheckResult` type. A check result should tell us the check that was run, the result of the check (`pass` or `fail`) and a timestamp when the check was run.

    abstract type AbstractCheckResult end

    struct CheckResult
        check_label::String
        check_result::String
        check_run_at::Dates.DateTime
        
        function CheckResult(
            check_label::String,
            check_result::String,
            check_run_at::Union{Dates.DateTime,Nothing} = nothing
        )
            input_array = [check_label]
            # Ensure check_result is pass or fail.
            if check_result ∈ ["pass","fail"]
                push!(input_array, check_result)
            else
                ArgumentError("check_result string argument must be one of [\"pass\", \"fail\"]")
            end
            # check_run_at of `nothing` should use current timestamp.
            input_check_run_at = check_run_at isa Nothing ? Dates.now() : check_run_at
            push!(input_array, input_check_run_at)

            new(input_array...)
        end
    end

Here we have an inner constructor that makes sure the check result is either "pass" or "fail" and will default to the current timestamp when check_run_at is `nothing` or is left out. We don't need an outer constructor since we'll be making these check results during checking functions.

Now we're ready to make the logic for our checking function.

# Checking functions

We want to check if the applicant is 18 plus. But there are some _implicit_ checks we should run as well, such as making sure the application isn't in a terminal state and or has expired.

A check function is a functional abstraction we'll be using. It is any function that takes an `Application` argument and an `Applicant` argument and returns a `CheckResult` type.

Here's the full set of check functions---implicit and explicit---that we'll use in our endpoint.

    function application_not_expired(application::Application, args...)::CheckResult
        check_label = "application_not_expired"
        if application.expiry_deadline isa Nothing
            check_result = "pass"
        elseif application.expiry_deadline > Dates.now()
            check_result = "pass"
        else
            check_result = "fail"
        end
        return CheckResult(check_label, check_result)
    end

    function applicant_is_18_plus(application::Application, applicant::Applicant, args...)::CheckResult
        check_label = "applicant_is_18_plus"
        check_result = applicant.dob + Dates.Year(18) < application.created_at ? "pass" : "fail"
        return CheckResult(check_label, check_result)
    end

These are all straightforward. Note that some require both `application` and `applicant`. Others require only `application`. In either case, `application` should be given as the first argument.

We now have all the checks we're going to run in our `run_preliminary_checks` function. So let's edit that.

    function run_preliminary_checks(application::AbstractApplication, applicant::AbstractApplicant)::Vector{AbstractCheckResult}
        checks_set = [
            application_not_expired,
            applicant_is_18_plus
        ]

        check_result_set = [f(application, applicant) for f in check_set]

        return check_result_set
    end

Here we've created an array with each of the checks we want to run. We run each of them and return the results as an array of `CheckResult` objects. This concludes the main decision logic portion. We still have to code up the ingestion and rendeing.

# Ingestion and rendering

The ingestion converts a Julia dictionary (identical to the JSON) into the abstractions used by the decision logic. The rendering does the opposite.

## Ingestion
We know that the JSON `application` object should represent the `Application` type. And we know the JSON `applicant` object (under `information`) should contain what we need for the `Applicant` type. Because we added outer constructors above that accept keyword inputs, handling this is pretty straightforward.

    const information_type_mapper = Dict(
        :applicant => Applicant
    )

    function ingest_payload(request_dict::Dict, args...; kwargs...)::Dict

        request_dict[:application] = Application(request_dict[:application]...)
        
        for k in keys(request_dict[:information])
            request_dict[:information][k] = information_type_mapper[k](request_dict[:information][k])
        end

        return request_dict
    end

We're using a trick here to make the ingestion work for any `information` field. We create a constant called `information_type_mapper` that maps the key used for the JSON object to the corresponding type constructor. This lets us use the key symbol itself to call the constructor function. We then splat that portion of the request dictionary into the constructor to create the object.

The result of this call is a dictionary that matches the format of the original, but instead of values for `Application` or each information type, we have the instantiated types.

To properly pipe this into the following function, we can use an anonymous function in `routes.jl`. It will now look like this:

    route("/prelimchecks", method = POST) do
    message = jsonpayload()
    message |> 
        GFLOApp.ingest_payload |> 
        (d -> GFLOApp.preliminary_check_set(d[:application],d[:information][:applicant])) |> 
        GFLOApp.update_application |>
        GFLOApp.render_response |> 
        json

    end



## Rendering

The rendering function takes the vector of check results and converts them to a dictionary that can be converted to a JSON. It's worth noting that Dictionaries and (JSONs in general) the order of objects is not guaranteed (but the order within arrays _is_ guaranteed).

Here's the updated code for the `render_response` function.

    function render_response(check_result_list::Vector{AbstractCheckResult})::Dict
        out_dict = Dict()
        # Helper function to convert a type instance to a dictionary.
        typedict(x) = Dict(fn=>getfield(x, fn) for fn ∈ fieldnames(typeof(x)))
        # Convert each instance of CheckResult to a dictionary then assign the array to the `check_result_list` object in the dictionary.
        # USEFUL NOTE: The when being rendered to the Genie `json` function, the `JSON3` package is used, which automatically converts dates and datetimes.
        #              So basically, we don't need to re-format dates or datetimes as strings explicitly.
        out_dict[:check_result_list] = [typedict(check_result) for check_result in check_result_list]
        return out_dict
    end

That should be all we need.