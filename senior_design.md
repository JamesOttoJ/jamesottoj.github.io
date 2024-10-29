---
layout: default
title: Senior Design
nav_order: 2
---

**Summary:** The project I worked on was a C Code Critiquer (CCC). The CCC was designed to help students develop better code in the various C-focused classes at Iowa State. This is done with both static and dynamic code analysis. The static analysis will look at the source code, develop and Abstract Syntax Tree (AST), and use regex to find stylistic and functional problems. On the dynamic analysis side, professors can submit unit tests to test for specific behavior in the functions. Feedback will then be generated using a combination of the static and dynamic analysis findings. Because the team's advisor for this project is a professor for the embedded systems class, extra attention is being put on specific problems that students in the class face. This leads to additional requirements regarding the datasheet used in class also being put in the code analysis.

**My Role:** As for what I brought to the project, I have an interest in lower-level programming and knowledge in cybersecurity. This allowed me to help secure the application wrapper and solve complex problems in the analysis of the programs itself.

**My Analysis:** For the application, I did an initial look at the source code to identify the information flow and find possible problems with the handling of data. In this, I was able to find very little problems. There was good session handling, credential privacy, and query handling. The main problems came in the handling of dynamic code analysis. The previous solution just created a folder and ran code inside it. This has two major flaws. First, it assumes things about the runner that may or may not be true (I personally had problems with compiler differences). Second, it can allow the code to access the parent data and modify it. These problems can be solved quite simply using a docker container. The Dockerfile will ensure that a consistent environment is created, and it naturally creates a separation between it and the rest of the host.

**Takeaways:** This project taught me a lot about both compiler functions and securely running arbitrary code. Both of these skills are useful for my career goals in analyzing malware to prevent and respond to incidents as static and dynamic analysis are both important for succeeding in the role.