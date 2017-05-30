---
title: "Prereq: Defining a Controlled Vocabulary"
keywords: Customize Metadata
categories: How to Do All the Things
permalink: customize-metadata-controlled-vocabulary.html
folder: hydra/how-to/customize_metadata/hyrax_1.0.0.rc1/controlled-vocabulary.md
sidebar: home_sidebar
tags: [development_resources]
---

NOTE: Please note that this documentation applies to Hyrax 1.0.0.rc1.

This tutorial assumes that you generated a work type name GenericWork.  If you used a different name, substitute that name for all occurrences of GenericWork and generic_work.

---

Creating controlled vocabularies is not required for your app, but we will be using one in this tutorial.  So we'll go ahead and create it now.

Controlled vocabularies can be created in your app and then accessed through the [Questioning Authority (QA)](https://github.com/projecthydra-labs/questioning_authority) gem.  These instructions show a simple way to create your own controlled vocabulary.  To get information on accessing external authorities and details on creating local sub-autorities, see the QA README, especially the [Local Sub-Authorities](https://github.com/projecthydra-labs/questioning_authority#local-sub-authorities) section.

Once the controlled vocabulary is created, you can use it for autocomplete.  See [Modifying the Edit Form]() -> [Customizing the form field](customize-metadata-edit-form.html#customizing-the-form-field) -> [For a controlled vocabulary](customize-metadata-edit-form.html#for-a-controlled-vocabulary-required).

## Create a vocabulary

Authorities are defined as a yml file in `config/authorities`.  Here we will define a departments controlled vocabulary.

```ruby
## config/authorities/departments.yml
terms:
  - id: eng
    term: English
  - id: hst
    term: History
  - id: ltn
    term: Latin
  - id: zoo
    term: Zoology
```

## Create a service to load the vocabulary

A service is required to set up Questioning Authority to return all the values, which will be used to populate the selection list, and a single value given an id, which will be used to show the value instead of the id on the show page. 

```ruby
# services/departments_service.rb
module DepartmentsService
  mattr_accessor :authority
  self.authority = Qa::Authorities::Local.subauthority_for('departments')

  def self.select_all_options
    authority.all.map do |element|
      [element[:label], element[:id]]
    end
  end

  def self.label(id)
    authority.find(id).fetch('term')
  end
end
```

---

<p><a href="customize-metadata-generate-work-type.html"><button type="button" class="btn btn-primary">Prev: Prereq: Generating a Work Type</button></a>  <a href="customize-metadata-labels.html"><button type="button" class="btn btn-primary">Next: Labels and Help Text</button></a></p>
