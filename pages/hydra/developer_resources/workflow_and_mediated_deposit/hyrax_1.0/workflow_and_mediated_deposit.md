---
title: "Workflow and Mediated Deposit in Hyrax 1.x"
keywords: Mediated Deposit, Workflow, Sipity
categories: Workflow
permalink: workflow_and_mediated_deposit.html
folder: hydra/developer_resources/workflow_and_mediated_deposit/hyrax_1.0/workflow_and_mediated_deposit.md
sidebar: home_sidebar
tags: [development_resources]
---

Note: This page borrows heavily, and hopes to update, the information contained on [the Sufia Mediated Deposit page](https://github.com/samvera/sufia/wiki/Mediated-Deposit-Workflow).

## Why mediated deposit?

In the hydra community, there are many people who would like to be able to ingest an item and make sure it doesn't go live until the item and its metadata have been thoroughly reviewed. This is where *mediated deposit* comes in. In Hyrax, a user can describe groups of steps they want to take to have an item "properly" reviewed. Hyrax uses *workflows* to describe these groups of steps to review an item that was ingested.

## What is a workflow?

In Hyrax, a workflow is a state machine generated pragmatically by utilizing work done in a gem called [Sipity](https://github.com/ndlib/sipity). Each workflow can look different depending on who has designed it. By taking this info into account, it is now possible to generate any workflow a user would like by describing it in a JSON format.

## How do I use the mediated deposit workflow that ships with Hyrax?

Hyrax comes with a one step approval workflow. You can see the file in `config/workflows/mediated_deposit_workflow.json`.

Make sure you've imported all of the files in the workflows directory:
```ruby
rake hyrax:workflow:load
```
In order to use the mediated deposit workflow, you first need to create a new Admin Set and and assign it to use the workflow. You can do this either via the UI, or programatically.

### Via the UI

  Make sure you have an admin user. Look at your `config/role_map.yml` file and ensure that the user you want to have admin priviliges is defined under an `admin` section, like this:

  ```
  development:
    archivist:
      - archivist1@example.com
    admin:
      - admin@example.com
  ```

  Now, if you log in via the admin user, you should see an menu called "Administration" in the top left corner. Under it, you should see a menu item on the left called "Administrative Sets." Under that menu, you can create a new administrative set. Give it a name and description and click save. Now you can assign it a workflow under the "Workflow" tab that will appear. You will also need to give it participants under the `participants` tab.

  You will also need to use the `workflow --> roles` menu to add users as approvers for the workflow.

### Programatically

Here is a console walk through of how to set up workflow programatically. This is a good idea if you have many workflows, or complex setup, that you need to re-create over and over again and don't want to make a person do it manually.

#### Make a new AdminSet, tell it to use the `one_step_mediated_deposit` workflow
```ruby
  a = AdminSet.new
  a.title = ["Emory ETDs"]
  a.save
  Hyrax::AdminSetCreateService.call(admin_set: a, creating_user: User.first!)

  # this will tell you what workflow was assigned. Probably the default.
  a.active_workflow

  # Get all the workflows available for this AdminSet's permission_template
  available_workflows = a.permission_template.available_workflows

  => #<ActiveRecord::Associations::CollectionProxy [#<Sipity::Workflow id: 1, name: "default", label: "Default workflow", description: "A single submission step, default workflow", created_at: "2017-04-25 18:21:41", updated_at: "2017-04-25 18:21:43", permission_template_id: 1, active: true, allows_access_grant: true>, #<Sipity::Workflow id: 2, name: "one_step_mediated_deposit", label: "One-step mediated deposit workflow", description: "A single-step workflow for mediated deposit in whi...", created_at: "2017-04-25 18:21:41", updated_at: "2017-04-25 18:21:43", permission_template_id: 1, active: nil, allows_access_grant: false>]>

  mediated_workflow = a.permission_template.available_workflows.where(name: "one_step_mediated_deposit").first

  Sipity::Workflow.activate!(permission_template: a.permission_template, workflow_id: mediated_workflow.id)

  # Now calling #active_workflow should give you the mediated deposit workflow
  a.active_workflow

  => #<Sipity::Workflow id: 2, name: "one_step_mediated_deposit", label: "One-step mediated deposit workflow", description: "A single-step workflow for mediated deposit in whi...", created_at: "2017-04-25 18:21:41", updated_at: "2017-04-25 18:44:48", permission_template_id: 1, active: true, allows_access_grant: false>
```
The AdminSet will be visible on the "relationships" tab of the deposit form. If you only have one admin set, it will be set by default. In order for it to work, a non-admin user must deposit a work, and an admin user must approve it.

<ul class="warning"><li>Calling `#activate!` on a Sipity::Workflow a second time will *deactivate* that workflow. If you want your workflow setup script to be idempotent, you should add a check to see if it is already activated.</li></ul>

#### Add a user to a workflow

```ruby
# First, you need a user
u = ::User.find_or_create_by(email: approver_email)
u.password = "123456"
u.save

# Then, you need to transform your User object into a Sipity::Agent
approving_agent = u.to_sipity_agent

workflow = a.active_workflow

# Get the Sipity::Role for approving
approval_role = Sipity::Role.find_by!(name: 'approving')

# Grant this one user the approving role for the EmoryETDs AdminSet
workflow.update_responsibilities(role: approval_role, agents: approving_agent)

```
<ul class="warning"><li>Calling `#update_responsibilities` on a workflow will add any agents you pass in the agents parameter, but it will also REMOVE any agents who are not in the agents parameter.</li></ul>

## How can an application set up workflow programatically?
 One pattern being used successfully is to put your workflow setup into its own class (e.g., `lib/workflow_setup.rb`) and give it a method (e.g., `setup`) that will configure everything as expected. That way you can follow best practice and make sure you are writing tests for it. Then, put something like this in `db/seeds.rb`:
```ruby
require 'workflow_setup'
w = WorkflowSetup.new
w.setup
```
Putting this code in `db/seeds.rb` means it will be called at the end of `bin/setup`, and you can call it anytime via `rake db:seed`

## How do I test that all of this is working correctly?

One approach is to make a capybara feature test for your approval workflow. You can log in
as different users, and have those users perform workflow roles. Here is an example:
```ruby
# Check the ETD was assigned the right workflow
etd = Etd.where(title: [title]).first
expect(etd.active_workflow.name).to eq "emory_one_step_approval"
expect(etd.to_sipity_entity.reload.workflow_state_name).to eq "pending_approval"

# Check workflow permissions for depositing user
available_workflow_actions = Hyrax::Workflow::PermissionQuery.scope_permitted_workflow_actions_available_for_current_state(user: user, entity: etd.to_sipity_entity).pluck(:name)
expect(available_workflow_actions.include?("mark_as_reviewed")).to eq false
expect(available_workflow_actions.include?("approve")).to eq false
expect(available_workflow_actions.include?("request_changes")).to eq false
expect(available_workflow_actions.include?("comment_only")).to eq false
expect(available_workflow_actions.include?("hide")).to eq false
expect(available_workflow_actions.include?("unhide")).to eq false

# Check notifications for depositing user
visit("/notifications?locale=en")
expect(page).to have_content "#{title} (#{etd.id}) was deposited by #{user.email} and is awaiting approval."

# Check notifications for approving user
logout
approving_user = User.where(email: "candleradmin@emory.edu").first
login_as approving_user
visit("/notifications?locale=en")
expect(page).to have_content "#{title} (#{etd.id}) was deposited by #{user.email} and is awaiting approval."

# Check workflow permissions for approving user
available_workflow_actions = Hyrax::Workflow::PermissionQuery.scope_permitted_workflow_actions_available_for_current_state(user: approving_user, entity: etd.to_sipity_entity).pluck(:name)
expect(available_workflow_actions.include?("mark_as_reviewed")).to eq false # this workflow step should only exist for Laney
expect(available_workflow_actions.include?("approve")).to eq true
expect(available_workflow_actions.include?("request_changes")).to eq true
expect(available_workflow_actions.include?("comment_only")).to eq true
expect(available_workflow_actions.include?("hide")).to eq true
expect(available_workflow_actions.include?("unhide")).to eq true

# Last superuser should have all workflow options available. (First superuser gets these by virtue of owning the admin sets.)
expect(w.superusers.count).to be > 1 # This test is meaningless if there is only one superuser
available_workflow_actions = Hyrax::Workflow::PermissionQuery.scope_permitted_workflow_actions_available_for_current_state(user: w.superusers.last, entity: etd.to_sipity_entity).pluck(:name)
expect(available_workflow_actions.include?("mark_as_reviewed")).to eq false # this workflow step should only exist for Laney
expect(available_workflow_actions.include?("approve")).to eq true
expect(available_workflow_actions.include?("request_changes")).to eq true
expect(available_workflow_actions.include?("comment_only")).to eq true
expect(available_workflow_actions.include?("hide")).to eq true
expect(available_workflow_actions.include?("unhide")).to eq true

# The approving user marks the etd as approved
subject = Hyrax::WorkflowActionInfo.new(etd, approving_user)
sipity_workflow_action = PowerConverter.convert_to_sipity_action("approve", scope: subject.entity.workflow) { nil }
Hyrax::Workflow::WorkflowActionService.run(subject: subject, action: sipity_workflow_action, comment: nil)
expect(etd.to_sipity_entity.reload.workflow_state_name).to eq "approved"

# Check notifications for approving user
visit("/notifications?locale=en")
expect(page).to have_content "Deposit #{title} has been approved"
expect(page).to have_content "#{title} (#{etd.id}) has been approved by"

# Check notifications for depositor again
logout
login_as user
visit("/notifications?locale=en")
expect(page).to have_content "Deposit #{title} has been approved"
expect(page).to have_content "#{title} (#{etd.id}) has been approved by"
```

## How do I set up an app to use a given workflow by default?

coming soon...

## How do I diagram and configure a new workflow?

coming soon ...
