<a id="us-projects.fork"></a>
# US-projects.fork: Fork demo project into user workspace

```gherkin
@US-projects.fork @UC-projects.fork @P0 @organization @fork
Feature: US-projects.fork Fork demo project into user workspace

  Background:
    Given user is authenticated as "Knowledge Engineer"
    And a public project "VEDO Demos/Organization" exists

  Scenario: Knowledge Engineer forks a demo project
    When user opens "VEDO Demos/Organization"
    And clicks "Fork"
    Then a new private project is created in user's personal space
    And the new project contains a copy of the "Organization" ontology
    And the new project has upstream_project_id = "VEDO Demos/Organization"
    And the user becomes Owner of the new project

  Scenario: User forks a private project without access
    When user attempts to fork a private project they don't have access to
    Then the system returns 403 Forbidden
    And no new project is created
```
