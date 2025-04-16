# Run Appium Mobile Automation Tests on GitHub Actions

This workflow automates running Appium-based mobile automation tests on a GitHub Actions runner. It allows you to trigger tests manually for specific projects defined in the workflow inputs.

## Workflow Details

### Trigger

This workflow is triggered manually using `workflow_dispatch`.

### Inputs

When manually triggering the workflow, you will be prompted to provide the following input:

- **Project**:

  - **Description**: The specific project for which to run the mobile automation tests.
  - **Required**: Yes
  - **Default**: `project-1`
  - **Type**: `choice`
  - **Options**:
    - `project-1`
    - `project-2`

  This input allows you to target your test execution to a specific project, likely used for filtering tests based on tags or directory structure within your test suite.

### Jobs

The workflow contains a single job named `mobile-automation`.

- **Runs On**: `ubuntu-latest` - This specifies that the job will run on the latest version of Ubuntu Linux, a common and reliable environment for CI/CD pipelines.

### Steps

The `mobile-automation` job consists of the following steps:

1.  **Checkout repository**:

    - Uses the `actions/checkout@v4` action to clone your repository onto the runner. This is essential to access your test code and application under test.

2.  **Cache Results Data DB**:

    - Uses the `actions/cache@v3` action to cache the `result_data/` directory.
    - **Key**: `result-data-DB` - This is the primary key used to save and retrieve the cache.
    - **Restore Keys**: `result-data-DB` - If an exact match for the key isn't found, it will attempt to restore from these fallback keys (in this case, just the primary key again).
    - This step likely caches a database used by your Robot Framework dashboard to store historical test results, potentially speeding up subsequent runs.

3.  **Cache Node modules**:

    - Uses the `actions/cache@v3` action to cache the `~/.npm` directory.
    - **Key**: `${{ runner.os }}-node` - This key includes the operating system of the runner, ensuring that caches are not shared between different OS environments.
    - **Restore Keys**: `${{ runner.os }}-node` - Similar to the previous cache step, this provides fallback keys for cache restoration.
    - This step caches Node.js modules installed via `npm`, significantly reducing the time needed to install Appium and its dependencies in subsequent runs.

4.  **Setup Python**:

    - Uses the `actions/setup-python@v5` action to set up the Python environment.
    - **python-version**: `'3.12.4'` - Specifies the exact Python version to use.
    - **cache**: `'pip'` - Enables caching of pip packages, speeding up the installation of Python dependencies.

5.  **Install dependencies**:

    - Runs the command `pip install -r requirements.txt` to install the Python dependencies listed in your `requirements.txt` file. This likely includes libraries needed for your Robot Framework tests or other utilities.

6.  **Install Appium**:

    - Runs a series of commands to install Appium and its necessary drivers:
      - `npm install -g appium@v2.17.1`: Installs the specified version of Appium globally using npm.
      - `appium driver install uiautomator2`: Installs the `uiautomator2` driver, which is commonly used for testing Android applications.
      - `appium -v`: Prints the installed Appium version to the console for verification.
      - `appium &>/dev/null &`: Starts the Appium server in the background. The output is redirected to `/dev/null` to keep the logs clean.

7.  **Enable KVM**:

    - Runs commands to enable Kernel-based Virtual Machine (KVM) on the runner. This is often necessary for running Android emulators efficiently.
      - `echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' | sudo tee /etc/udev/rules.d/99-kvm4all.rules`: Adds a udev rule to grant broader permissions to the KVM device.
      - `sudo udevadm control --reload-rules`: Reloads the udev rules.
      - `sudo udevadm trigger --name-match=kvm`: Triggers the newly added rule for the KVM device.

8.  **AVD cache**:

    - Uses the `actions/cache@v3` action to cache Android Virtual Devices (AVDs) and ADB-related files.
    - **id**: `avd-cache` - Assigns an ID to this cache step, allowing other steps to refer to its outputs.
    - **path**:
      - `~/.android/avd/*`: Caches the AVD configurations.
      - `~/.android/adb*`: Caches ADB server and key files.
    - **key**: `avd-30` - The cache key, likely indicating the Android API level of the cached AVD.

9.  **create AVD and generate snapshot for caching**:

    - This step is conditionally executed only if the AVD cache was not hit (`if: steps.avd-cache.outputs.cache-hit != 'true'`).
    - Uses the `reactivecircus/android-emulator-runner@v2` action to create an Android emulator.
    - **api-level**: `30`: Specifies Android API level 30.
    - **target**: `google_apis`: Uses a build with Google APIs included.
    - **arch**: `x86_64`: Specifies the architecture for the emulator.
    - **force-avd-creation**: `false`: Prevents forced creation if an AVD with the same configuration already exists.
    - **emulator-options**: `-no-window -gpu swiftshader_indirect -noaudio -no-boot-anim -skin 540x1200 -dpi-device 280`: Configures the emulator to run headless (without a graphical window), use software rendering for the GPU, disable audio and boot animation, and set a specific screen resolution and DPI.
    - **disable-animations**: `true`: Disables animations within the emulator for faster execution.
    - **script**: `echo "Generated AVD snapshot for caching."`: A simple script that runs after the AVD is created, likely indicating the purpose of this step. This step focuses on creating a base AVD and potentially saving a snapshot for faster startup in subsequent runs.

10. **Run Robot Framework Appium Tests**:

    - Uses the `reactivecircus/android-emulator-runner@v2` action again to run the Robot Framework tests.
    - **continue-on-error**: `true`: Allows the workflow to continue even if this step fails.
    - **api-level**, **target**, **arch**, **force-avd-creation**, **disable-animations**, and **emulator-options**: These options are the same as the previous AVD creation step, ensuring a consistent emulator environment.
    - **emulator-options**: `-no-snapshot-save -no-window -gpu swiftshader_indirect -noaudio -no-boot-anim -skin 540x1200 -dpi-device 280`: Importantly, `-no-snapshot-save` is added here, indicating that changes made during the test execution will not be saved to a snapshot. This ensures a clean state for each test run.
    - **script**:
      - `robot -d results --loglevel TRACE -i ${{ inputs.projects }} tests`: Executes the Robot Framework tests.
        - `-d results`: Specifies the directory to save the test results.
        - `--loglevel TRACE`: Sets the logging level to show detailed information.
        - `-i ${{ inputs.projects }}`: Includes tests based on the `projects` input value (likely using tags).
        - `tests`: Specifies the directory containing your Robot Framework test files.
      - `robotdashboard -o results/output.xml:${{ inputs.projects }} -d result_data/robot_result_database.db -n results/result_robot_dashboard.html -t "Dashboard Mobile Automation"`: Generates a Robot Framework dashboard report.
        - `-o results/output.xml:${{ inputs.projects }}`: Specifies the input XML report file and potentially associates it with the selected project.
        - `-d result_data/robot_result_database.db`: Specifies the database to store the dashboard data.
        - `-n results/result_robot_dashboard.html`: Specifies the output HTML file for the dashboard.
        - `-t "Dashboard Mobile Automation"`: Sets the title of the dashboard report.

11. **Upload Robot Framework Reports**:
    - Uses the `actions/upload-artifact@v4` action to upload the generated Robot Framework test reports.
    - **name**: `robot-reports`: Sets the name of the artifact.
    - **path**: `results/`: Specifies the directory containing the reports to be uploaded.

This workflow provides a comprehensive setup for running Appium mobile automation tests using Robot Framework on GitHub Actions. It includes steps for setting up the environment, caching dependencies and AVDs for faster execution, running tests for specific projects, generating a dashboard report, and uploading the test results as artifacts.
