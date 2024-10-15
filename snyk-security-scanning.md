# Snyk

### What is Snyk?

- Snyk is an application security platform that provides a complete view of all vulnerabilities in a source code repository.
- It offers insights into vulnerabilities in code you write (a.k.a 1st party code scanning), code you import (a.k.a 3rd party code scanning or SCA), Infrastructure as Code (IaC), and containers.

Scan results are published to the Snyk tool, which can be accessed at [Snyk](https://snyk.io/org/comcast-xvp/). It uses SSO to log in to the tool. Once logged in, you'll see all projects you have access to.

## Run Snyk Locally
 
### IDE Plugins

One of the best ways to prevent vulnerabilities in your applications is to fix vulnerabilities before you commit your code. Snyk provides easy-to-use plugins for some popular IDEs that can run a scan on the code in your IDE, informing you of detected vulnerabilities that you can fix prior to sharing that code with others.

Additionally, developers can integrate local IDEs with the Snyk application. Reference link: [Snyk IDE Plugins and Extensions](https://docs.snyk.io/scm-ide-and-ci-cd-integrations/snyk-ide-plugins-and-extensions)

### For macOS users:

1. Tap the Snyk repository:
    ```sh
    brew tap snyk/tap
    ```

2. Install Snyk:
    ```sh
    brew install snyk
    ```

3. Verify that Snyk has been installed correctly:
    ```sh
    snyk --version
    ```

4. Authenticate Snyk using your organization token:
    ```sh
    snyk auth
    ```
   Now redirecting you to our auth page, go ahead and log in,
   and once the auth is complete, return to this prompt and you'll
   be ready to start using snyk.


5. From the root directory of any repository, use the following command after authentication:
    ```sh
    snyk test --all-projects
   

#### References:

- [Snyk Self-Onboarding Through GitHub Cloud App](https://devsecdocs.r3.app.cloud.comcast.net/documentation/snyk-self-onboarding-thru-github-cloud-app.html)
- [Snyk Overview and Benefits](https://devsecdocs.r3.app.cloud.comcast.net/documentation/snyk-overview-and-benefits.html)