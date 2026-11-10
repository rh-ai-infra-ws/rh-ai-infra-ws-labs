# Workshop Workbench Setup

> **Note:** This step is **optional**. It is intended for participants who do not have the `oc` CLI installed locally or who want to explore additional features of the OpenShift AI Workbenches.

In the previous labs, you installed the OpenShift AI Operator and configured its components. In this section, you will set up a browser-based development environment using the **RHOAI Workbenches** feature. This provides a VS Code IDE directly in your browser with the `oc` and `kubectl` CLIs pre-installed, so you can follow all workshop exercises without any local tooling.

## Create a Data Science Project

A Data Science Project maps to an OpenShift namespace and provides the organizational boundary for your workbenches, pipelines, and model servers.

1. Open the **OpenShift AI Dashboard**.
   - From the OpenShift Web Console, click the **Application Launcher** (grid icon) in the top navigation bar and select **Red Hat OpenShift AI**.
2. In the left sidebar, click **Projects**.

![RHOAI Projects page](images/v35-workbench-projects.png)

3. Click **Create a project**.
4. Fill in the following details:
   - **Name:** `team-workbench`
   - **Description:** `Dedicated workspace for running VS Code Server / web-based IDE`

![Create project dialog](images/v35-workbench-create-project.png)

5. Click **Create**.

You should now see the `team-workbench` project dashboard with tabs for Workbenches, Pipelines, Deployments, Cluster storage, and more.

![Project dashboard](images/v35-workbench-project-dashboard.png)

## Create a Code Server Workbench

Now you will create a workbench using the Code Server image. This gives you a full VS Code experience in the browser.

1. Inside the `team-workbench` project, select the **Workbenches** tab and click **Create workbench**.
2. Configure the workbench with the following settings:

   | Field | Value |
   |-------|-------|
   | **Name** | `code-server` |
   | **Image selection** | **Code Server &#124; Data Science &#124; CPU &#124; Python 3.12** |
   | **Version selection** | **3.5** (code-server v4.106, Python v3.12) |

3. Leave all other settings at their default values.

![Create workbench form](images/v35-workbench-create-workbench.png)

4. Click **Create workbench**.

> **Note:** The workbench may take 1–2 minutes to start while the container image is pulled and the pod initializes. The status will transition from **Starting** to **Running**.

5. Wait until the workbench status shows **Running**.

![Workbench running](images/v35-workbench-running.png)

## Access the Workbench

1. Once the workbench is **Running**, click the **Open** link next to the workbench name.
2. If prompted, authenticate with your OpenShift credentials and authorize the access.
3. VS Code will load in a new browser tab.
4. Open a terminal by navigating to **Terminal** → **New Terminal** (or use the shortcut `Ctrl+Shift+C`).

![VS Code with terminal menu](images/v35-workbench-vscode-terminal.png)

## Verify the CLI Tools and Log In

From the terminal inside VS Code, verify that the required CLI tools are available and log in to the cluster.

1. Verify that `oc` is installed:

   ```bash
   oc version
   ```

2. Verify that `kubectl` is installed:

   ```bash
   kubectl version --client
   ```

3. At this point, the terminal is running as the workbench service account. To interact with the cluster as a user, log in with your credentials:

   ```bash
   oc login -u admin --server=<API_URL>
   ```

   ?> 📋 **The rendered command should look like this:** `oc login -u admin --server=<API_URL>`. If the `--server` value shows a placeholder in angle brackets instead of a URL, set the **Cluster Domain** at the top of the home page before running it.

4. Verify the connection:

   ```bash
   oc whoami
   ```

   This should now return `admin` instead of the service account identity.

![CLI verification and login](images/v35-workbench-cli-verify.png)

**Congratulations!** Your browser-based development environment is ready. You will use this workbench terminal for the CLI exercises in the remaining modules.
