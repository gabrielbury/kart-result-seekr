# Kart Result SEEKR
Is a project conceived to get a go kart racing result on demand.
The user select a kart track, the date and time of the race and the process is started.

## Use journey
- User access a web interface, containing a form with some filters.
- Fill the filters (racing track, date and time) and submit the form.
- The request is sent to the API.
- API generates a protocol id
- A docker container, with a custom app image is created.
- the api returns the generated protocol to the user
- The container runs an application that uses an autonomous (selenium like) to login into the racing track website, go to the results page, selects the date of the race, search in the results list the actual race (by the time), and:
  - access the qualifying result, print the page to pdf and downloads the file
  - access the race result, print the page to pdf and downloads the file
  - access the lap by lap times, print the page to pdf and downloads the file
- generates a zip file containing all three files and uploads it to a magalu cloud bucket
- clean up the downloaded files
- calls the api webhook with the link for the generated zip on bucket
- kills the container
- The user receives the link

## Stack
- react vite frontend
- node js (typescript) API
- node js crawler
- kubernetes
- web socket to receive the generated link?

## Task
You are a software architect, with large experience in distributed systems, security concerned and willed to deliver high quality professional scalable software solutions.
Generate a detailed planning, listing all the tasks and steps for build this complete solution.
In this planning, each piece of software or applications, must have its tasks well defined.
