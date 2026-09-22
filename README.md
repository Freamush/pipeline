# CI Pipeline with GitLab integration

This pipeline was created during one of my study projects. It was hosted on my self-hosted Jenkins runner and contains 5 stages.
Stack: Jenkins, PHPStan, Psalm, OWASP Dependency-Check, SonarQube

#Stage 1
This stage performs a checkout. During this stage, the latest code is pulled for the agent, and the GitLab status is updated to indicate the beginning of the process.

#Stage 2
This stage installs project dependencies via Composer. The build/ directory is recreated in order to ensure a clean state. The GitLab status is updated.

#Stage 3
This stage runs two static analysis tools, PHPStan and Psalm, in parallel in order to optimize time. Both tools publish their reports in JSON format.
Psalm runs with taint analysis enabled, so it can detect potential injection vulnerabilities. Findings from either tool mark the build as unstable, so the full result of the pipeline is visible later on without stopping it immediately.

#Stage 4
This stage scans dependencies (excluding vendor/) against the National Vulnerability Database (NVD) to detect known CVEs in third-party libraries.

#Stage 5
This stage runs static analysis via SonarQube and performs code quality checks. At this point, the pipeline waits for the Quality Gate decision on whether to pass or fail the build.

The final GitLab commit status is updated to reflect the overall pipeline success or failure.
