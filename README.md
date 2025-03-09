********Run the following command to set Docker's apt repository********

sudo apt update && \
sudo apt install ca-certificates curl gnupg && \
sudo install -m 0755 -d /etc/apt/keyrings && \
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
sudo chmod a+r /etc/apt/keyrings/docker.gpg && \
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null && \
sudo apt update

Install Docker by running this command:

sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

Clone the company's source code using git command:

git clone https://github.com/timpamungkasudemy/course-betamart-marketing

Go to the cloned source code directory

cd course-betamart-marketing

and see the content of the directory

ls

The source code directory already contains a Dockerfile. Build the Docker image using the Dockerfile.

sudo docker build -t betamart/marketing:1.0.0 .

Try to run the Docker image. The application runs on port 8888, thus you need to expose the port.

sudo docker run -d --name betamart-marketing -p 8888:8888 betamart/marketing:1.0.0

Ensure the application is running. The response of the following command must indicates that the application is up

curl http://localhost:8888/health



2nd step:
Check what are packages installed in the Docker image's Linux. Enter the Docker container's shell

sudo docker exec -ti betamart-marketing bash

The Docker image itself contains a shell (bash). While a shell can be used for troubleshooting, offering a shell as part of a container image potentially opens the door for attackers.

Even worse, the shell user is root, which means it has full operating-system-level access.

On the Docker shell, list the installed packages

apt list --installed

There are many Linux packages installed, including curl and wget which is the penetration tester's finding. Additionally, the more software inside a container image, the more vulnerabilities it may have.

Exit the Docker shell

exit

See the Docker image size

sudo docker images

The docker image size for betamart-marketing:1.0.0 is big (around 900MB). This indicates that the image might be bloated with unnecessary packages.

Check the Dockerfile

more Dockerfile

The Dockerfile uses base image golang:xxx (FROM golang:1.22.4). This base image actually contains a Go runtime.

The Go source code is compiled as a native executable (RUN CGO_ENABLED=0 GOOS=linux go build -o /betamart-marketing). Thus, the Go executable should no longer be needed to run the image as a container.



3rd step:

xample approach
To implement the recommendation, you can use the Linux distroless image offered by Google. Modify the Dockerfile.

nano Dockerfile

Remove all existing Dockerfile contents, and replace them with this one

# build stage
FROM golang:1.22.4 AS builder
WORKDIR /app
COPY go.mod .
COPY go.sum .
RUN go mod download
COPY *.go ./
RUN CGO_ENABLED=0 GOOS=linux go build -o /betamart-marketing
 
# final stage: use distroless image
FROM gcr.io/distroless/static-debian12:nonroot
WORKDIR /app/server
COPY --from=builder /betamart-marketing /app/server
CMD ["/app/server/betamart-marketing"]

This new Dockerfile builds a Docker image using a multi-stage build process.



Build Stage

Uses golang:1.22.4 for the Go environment.

Sets /app as the working directory.

Copies go.mod and go.sum, then runs go mod download to get Go dependencies.

Copies all .go files into the working directory.

Compiles the application with CGO_ENABLED=0 for a statically linked binary and GOOS=linux for Linux compatibility, outputting to /betamart-marketing.



Final Stage

Uses Google's minimal distroless image gcr.io/distroless/static-debian12:nonroot for security and size efficiency, running as a non-root user.

Sets /app/server as the working directory.

Copies the compiled binary (/betamart-marketing) from the build stage.

Specifies the binary to run (/app/server/betamart-marketing) when the container starts.

Save it by pressing CTRL + X (Windows) or Command + X (Mac)



Press y to save your change, and then confirm the prompt by pressing Enter (Windows) or Return (Mac)



Build the docker image using the new Dockerfile. Build as version 2.0.0

sudo docker build -t betamart/marketing:2.0.0 .

See the resulting image

sudo docker images

The 2.0.0 image size is a lot smaller (around 10MB)

Remove the existing betamart-marketing Docker container which uses the image version 1.0.0

sudo docker rm -f betamart-marketing

Try to run the Docker image version 2.0.0 to ensure the functionality works

sudo docker run -d --name betamart-marketing -p 8888:8888 betamart/marketing:2.0.0

Ensure the application is running. The response of the following command must indicate that the application is up

curl http://localhost:8888/health

Remove the existing betamart-marketing Docker container which uses the image version 2.0.0

sudo docker rm -f betamart-marketing

4th step:

Go to Docker Hub and sign up for a free account.


Find and remember your Docker username


Back to your Linux home directory

cd ~

Tag your local docker image that was previously built using your docker username. Give it name my-betamart-marketing with tag 2.0.0.

sudo docker tag betamart/marketing:2.0.0 <your-docker-username>/my-betamart-marketing:2.0.0

Sets an environment variable to enable Docker Content Trust (DCT) globally on your system.

export DOCKER_CONTENT_TRUST=1

Login to Docker Hub using your username and password

sudo docker login

Generate a new Docker signing key to your Docker username. The key enables the signing of container images, ensuring their integrity and publisher authenticity.

sudo docker trust key generate <your-docker-username>
For the passphrase, use MyDockerOne


Add a signer (a user authorized to sign images) to a specific Docker repository (my-betamart-marketing), using your public key.

sudo docker trust signer add --key <your-docker-username>.pub <your-docker-username> <your-docker-username>/my-betamart-marketing

For the root passphrase (blue box above), use MyDockerOne

For the new passphrase, use MyDockerTwo

Sign a specific Docker image tag, indicating the image has been verified and trusted.

sudo docker trust sign <your-docker-username>/my-betamart-marketing:2.0.0

For the passphrase, use MyDockerOne

Display detailed trust data for a specific Docker image tag. Ensure the image is signed.

sudo docker trust inspect --pretty <your-docker-username>/my-betamart-marketing:2.0.0
sudo docker trust inspect --pretty <your-docker-username>/my-betamart-marketing:2.0.0

The blue box above is your image digest.

Copy it.

You no longer need the signer and can remove it.

sudo docker trust signer remove <your-docker-username> <your-docker-username>/my-betamart-marketing

For the passphrase, use MyDockerTwo



