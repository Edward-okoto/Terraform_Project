# Terraform_Project
### Hosting a Dynamic Web App on AWS , with Terraform Module, Docker,Amazon ECR and ECS

### Instructions:
---

### **Task 1: Dockerization of Web App**
1. **Create a dynamic web application**:
   - Choose a technology you’re comfortable with (e.g., Node.js, Flask, Django).
   - For **Node.js**, follow these steps:
    1. Install [Node.js](https://nodejs.org/).
     - Confirm Node.js and node.js package manager has been installed.
       ```
       node -v
       npm -v
       ```
       ![](./img/t1.png)

    2. Initialize a project:
        ```bash
        mkdir webapp && cd webapp
        npm init -y
        ```

        When you run `npm init`, the `package.json` file is generated with a default `"test"` script like this:

        ```json
        "scripts": {
        "test": "echo \"Error: no test specified\" && exit 1"
        }
        ```

        This is just a placeholder indicating that no tests have been set up yet. To fix this and set up tests, you can replace the `"test"` script with a command to run your test framework.      
        ---

        ### **Step 1: Choose a Testing Framework**
        Select a testing framework for your project. Popular options for Node.js include:
        - **Jest** (easy and widely used)
        - **Mocha** (flexible and modular)

        ---

        ### **Step 2: Install the Framework**
        Use npm to install your chosen framework.

        For Jest:
        ```bash
        npm install --save-dev jest
        ```

        For Mocha:
        ```bash
        npm install --save-dev mocha
        ```
        ---

        ### **Step 3: Update `package.json`**
        Modify the `"scripts"` section in your `package.json` file to use the framework.

        For Jest:
        ```json
        "scripts": {
        "test": "jest"
        }
        ```

        For Mocha:
        ```json
        "scripts": {
        "test": "mocha"
        }
        ```

        ---

        ### **Step 4: Create Test Files**
        Add test files to your project. For example, create a `test/` directory, and inside it, create test files like `app.test.js`.

        For Jest:
        ```javascript
        test("Example test", () => {
        expect(1 + 1).toBe(2);
        });
        ```

        For Mocha:
        ```javascript
        const assert = require("assert");
        describe("Example Test", () => {
        it("should return true", () => {
         assert.strictEqual(1 + 1, 2);
         });
        });
        ```
        ---

        ### **Step 5: Run the Tests**
        Execute the tests using the `npm test` command:
        ```bash
        npm test
        ```
        ![](./img/t2.png)

    3. Install dependencies (e.g., Express framework):
        ```bash
        npm install express
        ```
    4. Create `index.js`:
        ```javascript
        const express = require("express");
        const app = express();
        app.get("/", (req, res) => res.send("Hello, World!"));
        app.listen(3000, () => console.log("App is running on port 3000"));
        ```
        ![](./img/t3.png)

2. **Write a `Dockerfile` to containerize the application**:
   Create a file named `Dockerfile` in the root of your project:

   ```Dockerfile
   FROM node:16  # Use official Node.js image
   WORKDIR /app  # Set working directory
   COPY package*.json ./  # Copy package files
   RUN npm install  # Install dependencies
   COPY . .  # Copy the rest of your app
   EXPOSE 3000  # Expose port 3000
   CMD ["node", "index.js"]  # Command to start app
   ```
   ![](./img/t4.png)

3. **Build and test the Docker image locally**:
   - Build the image:
     ```bash
     docker build -t webapp:latest ./webapp
     ```
     ![](./img/t5.png)

   - Run the container:
     ```bash
     docker run -p 3000:3000 webapp:latest
     ```
   - Access the app in your browser at `http://localhost:3000`.

        ![](./img/t6.png)
---

### **Task 2: Terraform Module for Amazon ECR**
1. **Set up Terraform project**:
   - Create a directory structure:
     ```bash
     mkdir -p terraform-ecs-webapp/modules/ecr
     ```
        ![](./img/t7.png)

2. **Write the ECR module (`modules/ecr/main.tf`)**:
   In the `modules/ecr/` directory, create `main.tf`:
   ```hcl
   resource "aws_ecr_repository" "webapp_repo" {
     name                 = "webapp-repo"
     image_tag_mutability = "MUTABLE"
     image_scanning_configuration {
       scan_on_push = true
     }
   }

   output "repository_url" {
     value = aws_ecr_repository.webapp_repo.repository_url
   }
   ```

3. **Outputs**:
   The  output will be used later to push Docker images to Amazon ECR.

---

### **Task 3: Terraform Module for ECS**
1. **Set up ECS module directory**:
   ```bash
   mkdir -p terraform-ecs-webapp/modules/ecs
   ```
   

2. **Write the ECS module (`modules/ecs/main.tf`)**:
   In the `modules/ecs/` directory, create `main.tf`:
   ```hcl
   resource "aws_ecs_cluster" "webapp_cluster" {
     name = "webapp-cluster"
   }

   resource "aws_ecs_task_definition" "webapp_task" {
     family = "webapp-task"
     container_definitions = jsonencode([
       {
         name    = "webapp-container",
         image   = var.image_uri,
         memory  = 512,
         cpu     = 256,
         essential = true,
       }
     ])
   }

   resource "aws_ecs_service" "webapp_service" {
     cluster        = aws_ecs_cluster.webapp_cluster.id
     task_definition = aws_ecs_task_definition.webapp_task.arn
     desired_count   = 1
   }

   variable "image_uri" {}
   ```

3. **Inputs for image URI**:
   Make sure `var.image_uri` is passed from the main configuration file.

---

### **Task 4: Main Terraform Configuration**
1. **Create the main `main.tf` file**:
   In the `terraform-ecs-webapp/` directory, write:
   ```hcl
   provider "aws" {
     region = "us-east-1"  # Change region if needed
   }

   module "ecr" {
     source = "./modules/ecr"
   }

   module "ecs" {
     source    = "./modules/ecs"
     image_uri = module.ecr.repository_url
   }
   ```
   ![](./img/t8.png)

2. **Initialize Terraform**:
   Run the following commands in the `terraform-ecs-webapp/` directory:
   ```bash
   terraform init
   ```
    ![](./img/t9.png)

---

### **Task 5: Deployment**
1. **Build Docker image**:
   Build the web app Docker image:
   ```bash
   docker build -t webapp .
   ```

2. **Push image to Amazon ECR**:
   - Authenticate with ECR:
     ```bash
     aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ECR_repository_uri>
     ```

     ![](./img/t10.png)


   - Tag and push the image:
     ```bash
     docker tag webapp:latest <ECR_repository_uri>:latest
     docker push <ECR_repository_uri>:latest
     ```

        ![](./img/t11.png)
        ![](./img/t12.png)
        ![](./img/t13.png)

3. **Apply Terraform configuration**:
   Deploy your infrastructure:
   ```bash
   terraform apply
   ```
   Confirm changes by typing `yes` when prompted.

4. **Access the Web App**:
   After deployment, find the ECS service’s public IP or DNS in the AWS Management Console or Terraform outputs. Open it in your browser to access the app.

---

