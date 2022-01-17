# Slack-and-Plumber-Part-One

The details of the codeset and plots are included in the attached Microsoft Word Document (.docx) file in this repository. 
You need to view the file in "Read Mode" to see the contents properly after downloading the same.

Plumber Package - A Brief Introduction
=======================================

This asset shows how plumber can be used to build a Slack slash command. The API is built on top of a simulated customer dataset that contains details about customer call history. The slash command provides access to customer status report as well as customer success rep reports directly from within Slack. The goal of this integration is to highlight the strengths of plumber and how it can be used to reliably and securely integrate R with other products and services.

Using
=======
Once you have created the slack app and the slash command as described in Getting Started, you can access the API from within the slack interface.

Commands
==========
    /cs help
    /cs status <customer_id>
    /cs rep <rep_name>
    /cs region <region_name>
        
Examples
Try the following examples using the /cs command with your slackbot.

    /cs status 10
    /cs rep Lovey Torp MD
    /cs region east


Simulated data
================
The API pulls data from a simulated customer dataset using the wakefield package and the charlatan package. You can use the following levels for rep, region, and id.

Details
========
Instead of registering a different command for each endpoint, the first argument provided to the slash command is the endpoint while the subsequent argument(s) (if necessary) provide additional data to be passed to the specified endpoint. This way, a single slash command serves multiple endpoints without polluting the slash command namespace.To access a customer status report, enter /cs status <id> in Slack, where id is a valid customer ID from the simulated data. The customer status report includes the customer name, total calls, date of birth, and a plot call totals for the last 20 weeks. The color of the message is an indication of customer health. Green indicates the customer has no issues while red indicates the customer has a high volume of calls, indicating a potential problem.Help for all available commands can be accessed by entering /cs help or simply /cs into Slack.

Getting Started
=================

In order to build a Slack app, you must have a Slack account and follow the directions for creating a Slack app. The app will be tied to a specific workspace, so select a Slack workspace you anticipate belonging to long term. By default, your app will only be available to this workspace, although it’s possible to expand access to the app later on.
Once the app has been created in Slack, create a new slash command through which the end user will interact with the app.Specific details for building slash commands can be found here. This will be a helpful reference through the remainder of the walk through.

Plumbing the API
=================  
If you haven’t already, install the plumber package via install.packages ("plumber"). The plumber.R file uses plumber to define all of the API filters and endpoints leveraged by the Slack app. Here, we’ll go through each piece of the API to describe the code and introduce helpful resources.

In this scenario, we’re building an API that interacts with a known request. That is, we must build the API so that it can properly handle the request that comes from Slack. This is different from building an API that others will write requests for because in this instance, we have no control over the request. Instead, the API must be designed to properly interact with the Slack request. In order to promote this type of development, it is helpful to know how Slack makes requests and what is contained in those requests. This section of the Slack documentation contains helpful details about the request Slack makes in response to a slash command. In short, the request contains a url encoded data payload containing details about the slash command that was invoked. An example request looks like the following:

    token=gIkuvaNzQIHg97ATvDxqgjtO
    &team_id=T0001
    &team_domain=example
    &enterprise_id=E0001
    &enterprise_name=Globular%20Construct%20Inc
    &channel_id=C2147483705
    &channel_name=test
    &user_id=U2147483697
    &user_name=Steve
    &command=/weather
    &text=94070
    &response_url=https://hooks.slack.com/commands/1234/5678
    &trigger_id=13345224609.738474920.8088930838d88f008e0
  
Due to the way plumber handles data from incoming requests, there are two methods we can use to access this data within the API. First, this data will be parsed and arguments matched to the functions defined in the API. So, we could write a function that takes user_name as an argument and the user_name value from the request data would be passed into the function by plumber automatically. The other method is to access the entire data of the request using req$postBody. With this information in mind, we are prepared to start creating our API filters and endpoints.

Setup
======  
    # Packages ----
    library(plumber)
    library(magrittr)
    library(ggplot2)

    # Data ----
    # Load sample customer data. IRL this would likely be housed in a database.
    sim_data <- readr::read_rds("data/sim-data.rds")

    # Config options ----
    # Base URL for API requests
    base_url <- config::get("base_url")

    # Utils ----
    encrypt_string <- function(string) {
      urltools::url_encode(safer::encrypt_string(paste(Sys.time(), string, sep = ";"),
                                                 key = Sys.getenv("SLACK_SIGNING_SECRET")))
    }

    plot_auth <- function(endpoint, time_limit = 5) {
      # Save current time to compare against endpoint time value
      current_time <- Sys.time()

      # Try to decrypt endpoint and extract user id
      tryCatch({
        # Decrypt endpoint using SLACK_SIGNING_SECRET
        decrypted_endpoint <- safer::decrypt_string(endpoint,
                                                    key = Sys.getenv("SLACK_SIGNING_SECRET"))
        # Split endpoint on ;
        endpoint_split <- unlist(strsplit(decrypted_endpoint, split = ";"))
        # Convert time
        endpoint_time <- as.POSIXct(endpoint_split[1])
        # Calculate time difference
        time_diff <- difftime(current_time, endpoint_time, units = "secs")

        # If more than 5 seconds have passed since the request was generated, then
        # error
        if (time_diff > time_limit) {
          "Unauthorized"
        } else {
          endpoint_split[2]
        }
      },
      error = function(e) "Unauthorized"
      )
    }

    #* @apiTitle CS Slack Application API
    #* @apiDescription API that interfaces with Slack slash command /cs
  
Here we setup the environment for the API by loading the appropriate packages and loading the simulated data. The config package is used to store parameters that change based on the location of the API (if it’s local or deployed on RStudio Connect).There are two utility functions that are defined here. Both are used to authorize requests to plot endpoints and will be described in greater detail later.We also provide the API title and description. Now, we’re ready to start defining the filters and endpoints of the API.

    @filter route-endpoint
    #* Parse the incoming request and route it to the appropriate endpoint
    #* @filter route-endpoint
    function(req, text = "") {
      # Identify endpoint
      split_text <- urltools::url_decode(text) %>%
        strsplit(" ") %>%
        unlist()

      if (length(split_text) >= 1) {
        endpoint <- split_text[[1]]

        # Modify request with updated endpoint
        req$PATH_INFO <- paste0("/", endpoint)

        # Modify request with remaining commands from text
        req$ARGS <- split_text[-1] %>% 
          paste0(collapse = " ")
      }

      if (req$PATH_INFO == "/") {
        # If no endpoint is provided (PATH_INFO is just "/") then forward to /help
        req$PATH_INFO <- "/help"
      }

      # Forward request 
      forward()
    }
                         
This filter is responsible for parsing the text field of the incoming request and routing the request to the appropriate endpoint. Additional details provided in text are added to the request object (req) as req$ARGS. This filter also routes authorized requests made to / to the /help endpoint. This way, someone in Slack can simply enter /cs to get help for the command.

    @filter logger
    #* Log information about the incoming request
    #* @filter logger
    function(req){
      cat(as.character(Sys.time()), "-", 
          req$REQUEST_METHOD, req$PATH_INFO, "-", 
          req$HTTP_USER_AGENT, "@", req$REMOTE_ADDR, "\n")

      # Forward request
      forward()
    }
                         
This filter is lifted straight from the plumber docs. It simply logs information about incoming requests and is helpful when troubleshooting API performance and behavior.

    @filter verify
    #* Verify incoming requests
    #* @filter verify
    function(req, res) {
      # Forward requests coming to swagger endpoints
      if (grepl("swagger", tolower(req$PATH_INFO))) return(forward())

      # Check for X_SLACK_REQUEST_TIMESTAMP header
      if (is.null(req$HTTP_X_SLACK_REQUEST_TIMESTAMP)) {
        res$status <- 401
      }

      # Build base string
      base_string <- paste(
        "v0",
        req$HTTP_X_SLACK_REQUEST_TIMESTAMP,
        req$postBody,
        sep = ":"
      )

      # Slack Signing secret is available as environment variable
      # SLACK_SIGNING_SECRET
      computed_request_signature <- paste0(
        "v0=",
        openssl::sha256(base_string, Sys.getenv("SLACK_SIGNING_SECRET"))
      )

      # If the computed request signature doesn't match the signature provided in the
      # request, set status of response to 401
      if (!identical(req$HTTP_X_SLACK_SIGNATURE, computed_request_signature)) {
        res$status <- 401
      } else {
        res$status <- 200
      }

      if (res$status == 401) {
        list(
          text = "Error: Invalid request"
        )
      } else {
        forward()
      }
    }
  
This filter is used to to confirm that incoming requests are indeed coming from Slack and not an unauthorized source. Details about authenticating Slack requests can be found in Slack’s documentation. Essentially, Slack provides a signing secret that is known to us (the app developers) and Slack. This signing secret is used in combination with request details to calculate a signature for each request. That signature is verified in this filter to ensure that the request came from Slack. Requests made to Swagger endpoints are immediately forwarded without being authorized so that the Swagger UI can still be rendered.

    @post /help
    #* Help for /cs command
    #* @serializer unboxedJSON
    #* @post /help
    function(req, res) {
      list(
        # response type - ephemeral indicates the response will only be seen by the
        # user who invoked the slash command as opposed to the entire channel
        response_type = "ephemeral",
        # attachments is expected to be an array, hence the list within a list
        attachments = list(
          list(
            title = "/cs help",
            fallback = "/cs help",
            fields = list(
              list(
                title = "/cs status customer_id",
                value = "Customer summary",
                short = TRUE
              ),
              list(
                title = "/cs rep rep_name",
                value = "CS rep summary",
                short = TRUE
              ),
              list(
                title = "/cs region region_name",
                value = "Region report",
                short = TRUE
              )
            )
          )
        )
      )
    }
  
This endpoint posts a message in Slack that provides help for using this specific slash command.

    @post /status
    # unboxedJSON is used b/c that is what Slack expects from the API
    #* Return a message containing status details about the customer
    #* @serializer unboxedJSON
    #* @post /status
    function(req, res) {
      # Check req$ARGS and match to customer - if no customer match is found, return
      # an error

      # TODO: Provide suggestions for names that are close in the event of mispellings

      customer_ids <- unique(sim_data$id)
      customer_names <- unique(sim_data$name)

      if (!as.numeric(req$ARGS) %in% customer_ids & !req$ARGS %in% customer_names) {
        res$status <- 400
        return(
          list(
            response_type = "ephemeral",
            text = paste("Error: No customer found matching", req$ARGS)
          )
        )
      }

      # Filter data to customer data based on provided id / name
      if (as.numeric(req$ARGS) %in% customer_ids) {
        customer_id <- as.numeric(req$ARGS)
        customer_data <- dplyr::filter(sim_data, id == customer_id)
        customer_name <- unique(customer_data$name)
      } else {
        customer_name <- req$ARGS
        customer_data <- dplyr::filter(sim_data, name == customer_name)
        customer_id <- unique(customer_data$id)
      }

      # Simple heuristics for customer status
      total_customer_calls <- sum(customer_data$calls)

      customer_status <- dplyr::case_when(total_customer_calls > 250 ~ "danger",
                                          total_customer_calls > 130 ~ "warning",
                                          TRUE ~ "good")

      # Build response
      list(
        # response type - ephemeral indicates the response will only be seen by the
        # user who invoked the slash command as opposed to the entire channel
        response_type = "ephemeral",
        # attachments is expected to be an array, hence the list within a list
        attachments = list(
          list(
            color = customer_status,
            title = paste0("Status update for ", customer_name, " (", customer_id, ")"),
            fallback = paste0("Status update for ", customer_name, " (", customer_id, ")"),
            # History plot

            image_url = paste0(base_url, 
                               "/plot/history?cust_secret=",
                               encrypt_string(customer_id)),
            # Fields provide a way of communicating semi-tabular data in Slack
            fields = list(
              list(
                title = "Total Calls",
                value = sum(customer_data$calls),
                short = TRUE
              ),
              list(
                title = "DoB",
                value = unique(customer_data$dob),
                short = TRUE
              )
            )
          )
        )
      )
    }
  
This endpoint returns a status update for the specified customer. The update includes customer name, total calls, date of birth, and a plot of weekly calls for the previous 20 weeks. The response is serialized as unboxed JSON so that it matches the format defined by Slack. Since the image_url used by Slack is accessed via a simple GET request, there is no baked in authentication. In order to prevent sensitive data being leaked through customer plots, we use the encrypt_string() function defined in the utils section of the API setup. This function encrypts the customer id along with the current datetime using the signing secret previously obtained from Slack. When this endpoint is invoked, the query string is decrypted and the datetime is compared to the current datetime. If more than 5 seconds have passed, the request is considered to be unauthorized.

    @get /plot/history
    #* Plot customer weekly calls
    #* @png
    #* @param cust_secret encrypted value calculated in /status endpoint
    #* @response 400 No customer with the given ID was found.
    #* @preempt verify
    #* @get /plot/history
    function(res, cust_secret) {
      # Authenticate that request came from /status
      cust_id <- plot_auth(cust_secret)

      # Return unauthorized error if cust_id is "Unauthorized"
      if (cust_id == "Unauthorized") {
        res$status <- 401
        stop("Unauthorized request")
      } else if (!cust_id %in% sim_data$id) {
        res$status <- 400
        stop("Customer id" , cust_id, " not found.")
      }

      # Filter data to customer id provided
      plot_data <- dplyr::filter(sim_data, id == cust_id)

      # Customer name (id)
      customer_name <- paste0(unique(plot_data$name), " (", unique(plot_data$id), ")")

      # Create plot
      history_plot <- plot_data %>%
        ggplot(aes(x = time, y = calls, col = calls)) +
        ggalt::geom_lollipop(show.legend = FALSE) +
        theme_light() +
        labs(
          title = paste("Weekly calls for", customer_name),
          x = "Week",
          y = "Calls"
        )

      # print() is necessary to render plot properly
      print(history_plot)
    }
  
This endpoint returns a plot of the call history for the given customer. One challenge with this endpoint is that we have no control over the request that’s made, so it is difficult to authenticate the incoming request (ie, we can’t send some secret with the request and verify against it). This endpoint is used in the messages we return to Slack, and Slack just views this as an image URL to which it makes a GET request. In order to ensure that this works as expected, #* @preempt verify is added to the definition of this endpoint so that the verify filter doesn’t apply here. As previously mentioned, this endpoint is secured using plot_auth(), which decrypts the query string and checks the timestamp to ensure that this request is made within 5 seconds of a request to /status. This effectively places an expiration date on calls to this endpoint. The default expiration is 5 seconds from when the call is made to /status.

    @post /rep
    #* Get summary of rep performance
    #* @serializer unboxedJSON
    #* @post /rep
    function(req, res) {
      # Check to ensure rep exists in data
      if (!req$ARGS %in% unique(sim_data$rep)) {
        return(
          list(
            response_type = "ephemeral",
            text = paste("Error: No rep found matching", req$ARGS)
          )
        )
      }

      rep_data <- dplyr::filter(sim_data, rep == req$ARGS)
      n_clients <- length(unique(rep_data$name))

      list(
        # response type - ephemeral indicates the response will only be seen by the
        # user who invoked the slash command as opposed to the entire channel
        response_type = "ephemeral",
        # attachments is expected to be an array, hence the list within a list
        attachments = list(
          list(
            title = paste0("Rep: ", req$ARGS),
            fallback = paste0("Rep: ", req$ARGS),
            fields = list(
              list(
                title = "Total Clients",
                value = n_clients,
                short = TRUE
              ),
              list(
                title = "Calls / Client",
                value = sum(rep_data$calls) / n_clients,
                short = TRUE
              )
            )
          )
        )
      )
    }
  
This endpoint returns details about a specific rep’s performance, specifically total clients and calls / client for that rep.

    @post /region
    #* Summary of region performance
    #* @serializer unboxedJSON
    #* @post /region
    function(req, res) {
      # Check to ensure provided region value exists in data
      if (!tolower(req$ARGS) %in% tolower(unique(sim_data$region))) {
        return(
          list(
            reponse_type = "ephemeral",
            text = paste("Error: No region found matching", req$ARGS)
          )
        )
      }

      region_data <- dplyr::filter(sim_data, tolower(region) == tolower(req$ARGS))

      list(
        response_type = "ephemeral",
        attachments = list(
          list(
            title = paste0("Region: ", req$ARGS),
            fallback = paste0("Region: ", req$ARGS),
            image_url = paste0(base_url, "/plot/region?region_name=",
                               encrypt_string(tolower(req$ARGS))),
            fields = list(
              list(
                title = "Total Clients",
                value = length(unique(region_data$name)),
                short = TRUE
              )
            )
          )
        )
      )
    }
                     
This endpoint posts a Slack message that contains a plot of the trend for a given region. This plot is secured using the same mechanism as /plot/history.

    @get /plot/region/<region_name>
    #* Plot region data
    #* @png
    #* @param region_name Name of region to be plotted
    #* @preempt verify
    #* @get /plot/region
    function(region_name, req, res) {
      region_name <- plot_auth(region_name)
      if (region_name == "Unauthorized") {
        res$status <- 401
        stop("Unauthorized request")
      } else if (!tolower(region_name) %in% tolower(sim_data$region)) {
        res$status <- 400
        stop("Region " , region_name, " not found.")
      }

      region_plot <- sim_data %>% 
        dplyr::filter(tolower(region) == tolower(region_name)) %>% 
        ggplot(aes(x = time, y = calls)) +
        geom_jitter(alpha = .3) +
        geom_smooth(se = FALSE) +
        theme_light() +
        labs(
          title = paste("Call trend for", region_name, "Region"),
          x = "Week",
          y = "Calls"
        )

      print(region_plot)
    }
  
This endpoint creates a plot for a specific region’s performance. Much like the customer history plot, this endpoint uses #* @preempt verify so that the verify filter does not apply to this endpoint and it makes use of plot_auth() to ensure only authorized requests are responded to.

Running Locally
================  
Interacting with these APIs locally can be a bit of a challenge since most require data to be passed in the request body. It’s also a challenge to mimic the request as it is sent by Slack, especially when it comes to mimicking the authentication mechanism. While traditional tools like curl can be used, I’ve found that Postman is a powerful and easy to use tool for interacting with APIs. Postman can even leverage pre-request JavaScript code to mimic the authentication mechanism employed by Slack. For example, I use the following JS code to mimic Slack authentication in local testing:

    // Define function for creating URI string from data object
    // Lifted from https://stackoverflow.com/questions/14525178/is-there-any-native-function-to-convert-json-to-url-parameters
    function urlfy(obj) {
      return Object
      .keys(obj)
      .map(k => `${encodeURIComponent(k)}=${encodeURIComponent(obj[k])}`)
      .join('&');
    }

    // Set timestamp of request
    var date = new Date()
    var timestamp = date.getTime()
    pm.globals.set("SLACK_TIMESTAMP", timestamp);

    // Build rawBody using urlfy
    var rawBody = urlfy(request.data)
    var baseString = ["v0", timestamp, rawBody].join(":")
    // console.log(baseString)

    // Calculate signature
    var signature = ["v0=", CryptoJS.HmacSHA256(baseString, pm.globals.get("SLACK_SIGNING_SECRET"))].join('')
    //console.log(signature)

    // Set SLACK_SIGNATURE variable
    pm.globals.set("SLACK_SIGNATURE", signature)
  
The entire Postman collection I use for interaction with the API is contained in the postman-api-collection.json file. This collection can be imported into Postman and used to interact with the API either locally or remotely.

Deployment
===========  
As mentioned, this API is deployed on RStudio Connect on the colorado demo server. Deployment is done through the publish button in the RStudio IDE. A vanity URL was used and then passed into the Slack app settings so that Slack knows where to send requests.




