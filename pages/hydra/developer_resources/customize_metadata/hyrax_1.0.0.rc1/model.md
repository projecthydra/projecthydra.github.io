---
title: "Defining Metadata in the Model"
keywords: Customize Metadata
categories: How to Do All the Things
permalink: customize-metadata-model.html
folder: hydra/how-to/customize_metadata/hyrax_1.0.0.rc1/model.md
sidebar: home_sidebar
tags: [development_resources]
---

NOTE: Please note that this documentation applies to Hyrax 1.0.0.rc1.

This tutorial assumes that you generated a work type name GenericWork.  If you used a different name, substitute that name for all occurrences of GenericWork and generic_work.

---

The GenericWork class is generated with some default metadata, but we want to update it with our own metadata.  The generated version of the file looks like...

```ruby
# Generated via
#  `rails generate hyrax:work GenericWork`
class GenericWork < ActiveFedora::Base
  include ::Hyrax::WorkBehavior
  include ::Hyrax::BasicMetadata
  # Change this to restrict which works can be added as a child.
  # self.valid_child_concerns = []
  validates :title, presence: { message: 'Your work must have a title.' }
end
```

## Basic and Core metadata

* Basic metadata fields are defined in (app/models/concerns/hyrax/basic_metadata.rb)[https://github.com/projecthydra-labs/hyrax/blob/master/app/models/concerns/hyrax/basic_metadata.rb]
* Core metadata fields (that should never be removed) are defined in (app/models/concerns/hyrax/core_metadata.rb)[https://github.com/projecthydra-labs/hyrax/blob/master/app/models/concerns/hyrax/core_metadata.rb]


## Extend the model 

### To add a new single-value property
 
To define a property that has a single text value, add the following to the GenericWork model.
```ruby
  property :contact_email, predicate: ::RDF::Vocab::VCARD.hasEmail, multiple: false do |index|
    index.as :stored_searchable
  end
```

- It will be limited to a single value (set multiple: true  or leave off for multi-value, which is the default behavior)
- If included in the new/edit form, it will have `input type=text`  (There is a bit more configuration under section [Add the new single-value property to the new/edit form](#add-the-new-single-value-property-to-the-newedit-form) to have this included in the form.)
- By setting `index.as :stored_searchable`, values will be added to the solr_doc as contact_email_tesi indicating this field is English text (te), stored (s), indexed (i) 
  - See [Solr Schema](https://github.com/projecthydra/hydra-head/wiki/Solr-Schema) documentation for more information on dynamic solr field postfixes.
  - See [Solrizer::DefaultDescriptors](http://www.rubydoc.info/gems/solrizer/3.4.0/Solrizer/DefaultDescriptors) documentation for more information on values for `index.as`


| ![QUESTION](https://cloud.githubusercontent.com/assets/6855473/13064236/f2f04cbe-d41e-11e5-9674-e9a56a6326e6.png) | Are all values described in Solrizer::DefaultDescriptors supported within Hyrax property definitions? |


### To add a new multi-value property

To define a property that has multiple text values, add the following to the GenericWork model.

```ruby
  property :contact_phone, predicate: ::RDF::Vocab::VCARD.hasTelephone do |index|
    index.as :stored_searchable
  end
```

Expected behaviors for this property:
- Can have one or more values assigned.  NOTE: By default properties are multi-value.  You can also explicitly state this by adding `, multiple: true` before `do |index|`
- The remaining basic behaviors are the same as for single-value properties.  See more information under [Add the new single-value property to the model](#add-the-new-single-value-property-to-the-model) Expected behaviors.


### To add a new controlled vocabulary property

The process for adding a propery whose value comes from a controlled vocabulary is identical to that of the single and multi-value properties.  We will add a single-value controlled vocabulary field here so that it is available for use in later examples.

```ruby
  property :department, predicate: ::RDF::URI.new("http://lib.my.edu/departments"), multiple: false do |index|
    index.as :stored_searchable, :facetable
  end
```

Expected behaviors for this property:
- The behaviors are the same as for single-value properties because we set the property up to be single-value.  If this were multi-value, it would follow the behaviors of a multi-value field.


## The modified model

With all three properties added, the GenericWork now looks like:

```ruby
# Generated via
#  `rails generate hyrax:work GenericWork`
class GenericWork < ActiveFedora::Base
  include ::Hyrax::WorkBehavior
  include ::Hyrax::BasicMetadata
  # Change this to restrict which works can be added as a child.
  # self.valid_child_concerns = []
  validates :title, presence: { message: 'Your work must have a title.' }

  property :contact_email, predicate: ::RDF::Vocab::VCARD.hasEmail, multiple: false do |index|
    index.as :stored_searchable
  end

  property :contact_phone, predicate: ::RDF::Vocab::VCARD.hasTelephone do |index|
    index.as :stored_searchable
  end

  property :department, predicate: ::RDF::URI.new("http://lib.my.edu/departments"), multiple: false do |index|
    index.as :stored_searchable, :facetable
  end
end
```

---

<p><a href="customize-metadata-controller.html"><button type="button" class="btn btn-primary">Prev: Understanding the Controller</button></a>  <a href="customize-metadata-edit-form.html"><button type="button" class="btn btn-primary">Next: Modifying the Edit Form</button></a></p>
