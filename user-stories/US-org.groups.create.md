<a id="us-org.groups.create"></a>
# US-org.groups.create: Create group as a user

```gherkin
@US-org.groups.create @UC-org.groups.manage-group-lifecycle @P0 @organization @groups
Feature: US-org.groups.create Create group as a user

  Background:
    Given user is authenticated as "Owner"

  Scenario: User creates a private group with name and optional member invite
    When user navigates to "Groups" page
    And clicks "New group"
    Then the "Create group" dialog opens
    When user enters group name "Research Team"
    And selects visibility "Private"
    And optionally enters email "colleague@company.com" to invite a member
    And clicks "Create group"
    Then a new group "Research Team" is created with visibility "Private"
    And the user becomes Owner of the group
    And the group appears in the groups list
    And the group row shows member count = 1

  Scenario: User creates a subgroup under an existing group
    Given a group "Engineering" exists
    When user creates a new group with name "Frontend"
    And selects parent group "Engineering"
    Then a subgroup "Frontend" is created under "Engineering"
    And the subgroup inherits visibility from the parent group
    And expanding "Engineering" reveals "Frontend" as a child row

  Scenario: User attempts to create group with empty name
    When user opens "Create group" dialog
    And leaves group name empty
    And clicks "Create group"
    Then the dialog shows validation error "Group name is required"
    And no group is created

  Scenario: User selects public visibility for an open group
    When user creates a group with name "Open Research"
    And selects visibility "Public"
    Then the group "Open Research" is created with visibility "Public"
    And the group row shows a Globe visibility icon
    And any unauthenticated user can view the group page

  Scenario: User selects internal visibility
    When user creates a group with name "Internal Project"
    And selects visibility "Internal"
    Then the group "Internal Project" is created with visibility "Internal"
    And any authenticated user can view the group
    But external users cannot access it
```
