---
title: "Review-Driven Testing: Enhancing Code Quality, Upskilling Developers, and Promoting Testing Culture" 
date: 2024-01-23 12:00:00
---

In the world of software development, ensuring the quality and reliability of code is paramount. One popular approach to achieving this goal is Test-Driven Development (TDD), where developers write tests before writing the actual code. However, there are situations where code changes are made without accompanying automated tests. In such cases, a method known as "Review-Driven Testing" comes into play, not only to improve code quality and upskill developers but also to promote a testing culture within development teams.

## What is Review-Driven Testing?

Review-Driven Testing is a practical approach to adding automated tests to code changes that were initially developed without them. While traditional TDD focuses on writing tests before writing code, Review-Driven Testing involves writing tests after the code has been developed and submitted for review.

<!--more-->

## The Process in Review-Driven Testing:

1. **Code Review**: The process starts with a code review of changes that lack automated tests. The reviewer examines the code to understand its functionality and requirements.
1. **Writing Tests**: During the review, the reviewer writes the necessary automated tests for the code being reviewed. These tests are designed to validate that the code functions as intended and meets the specified requirements.
1. **Validation**: Upon successful validation, the code changes are approved, and the pull request is merged into the main codebase. While the code changes are now part of the project, it's important to note that they are not yet under automated test coverage. However, the subsequent steps will address this by introducing the necessary tests.
1. **Approval and Merge**: Upon successful validation, the code changes are approved, and the pull request is merged into the main codebase. This ensures that the code under review becomes part of the project with proper test coverage.
1. **Additional Pull Request**: After merging the code changes, a new pull request is opened by the reviewer. This pull request contains the automated tests created during the Review-Driven Testing process.
1. **Review by the Original Author**: The new pull request is assigned to the original code author for review. This step encourages collaboration and allows the original author to participate in the testing process and upskill their testing abilities. It also provides an opportunity for the original author to experience the benefits of automated testing.

## The Benefits of Review-Driven Testing

1. **Improved Code Quality**: Review-Driven Testing helps enhance code quality by ensuring that changes made to the codebase are thoroughly tested for correctness and adherence to requirements.
1. **Increased Test Coverage**: By adding tests after the initial development, the overall test coverage of the project increases. This contributes to a more robust and maintainable codebase.
1. **Collaboration and Skill Enhancement**: Review-Driven Testing fosters collaboration between developers and provides an opportunity for developers who may not be accustomed to writing tests to upskill and adopt good testing practices.
1. **Promoting a Testing Culture**: Through the positive exposure of having tests added during the Review-Driven Testing process, developers are more likely to include tests in their initial pull requests in the future. This method encourages a testing culture within development teams.

In conclusion, while Test-Driven Development (TDD) advocates writing tests before code, Review-Driven Testing offers a valuable alternative for enhancing code quality after development, upskilling developers, and fostering a culture of testing. So, whether you're reviewing code or having your own code reviewed, consider the benefits of Review-Driven Testing as a means to enhance the quality of your software, empower your team with valuable testing skills, and promote a culture of testing from the start.