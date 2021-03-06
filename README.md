# Conscript

Provides ActiveRecord models with `drafts` and `published` scopes, and the functionality to create draft instances and publish them.

Existing instances may have one or more draft versions which are initially created by duplication, including any required associations. A draft may then be published, overwriting the original instance.

Alternatively, draft instances may be created from scratch and published later.

The approach of the gem differs from others in that it does not create extra tables or serialise objects; instead it uses ActiveRecord's built-in scoping so that drafts have all of the same functionality as any other instance.

## Installation

Add this line to your application's Gemfile:

    gem 'conscript'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install conscript

## Usage

All models which wish to register for the draft must add two columns to their database schema, as illustrated in this example migration:

    class AddDraftColumns < ActiveRecord::Migration
      def self.up
        add_column :table_name, :draft_parent_id, :integer
        add_column :table_name, :is_draft, :boolean, default: false
        add_index :table_name, :draft_parent_id
        add_index :table_name, :is_draft
      end

      def self.down
        remove_column :table_name, :draft_parent_id
        remove_column :table_name, :is_draft
      end
    end

To use the drafts functionality, call `register_for_draft` in your ActiveRecord model:

    class Article < ActiveRecord::Base
      register_for_draft
    end

This registers `published` and `drafts` scopes on the model. You may use these scopes as with any other scope:

    Article.published

Or:

    Article.drafts

You'll therefore need to modify any existing code to use the `published` scope where you don't want to show drafts.


### Instance methods

Draft instances may optionally be created from existing instances:
    
    article = Article.first
    draft = article.save_as_draft!

Or you can create drafts from scratch:

    Article.new.save_as_draft!

You can access the original instance from the draft:

    draft.draft_parent == article # => true

And you can also access all of an instance's drafts:

    article.drafts # => [draft]

To determine whether an instance is a draft:

    article.is_draft? # => false
    draft.is_draft?   # => true

Publish a draft to overwrite the original instance:

    draft.publish_draft # returns the original, updated instance


### Options

By default, drafts are created with `ActiveRecord::Base#dup`, i.e. a shallow copy of all attributes, without any associations.

Options may be passed to the `register_for_draft` method, e.g:

    register_for_draft associations: :tags, ignore_attributes: :cached_slug

A list of options are below:

- `:associations` an array of `has_many` association names to duplicate. These will be copied to the draft and overwrite the original instance's when published. Deep cloning is possible thanks to the [`deep_cloneable`](https://github.com/moiristo/deep_cloneable) gem. Refer to the `deep_cloneable` documentation to get an idea of how far you can go with this. Please note: `belongs_to` associations aren't supported as these should be drafted separately.
- `:ignore_attributes` an array of attribute names which should _not_ be duplicated. Timestamps and STI `type` columns are excluded by default. Don't include association names here.
- `:allow_update_with_drafts` (`false`) whether to allow an instance to be updated if it has draft instances
- `:destroy_drafts_on_publish` (`true`) whether to destroy all other drafts for an instance when publishing a draft


### Callbacks

Two extra callbacks are made available for you to wrap bespoke behaviour around the draft lifecycle:

- `save_as_draft`
- `publish_draft`

You can call `set_callback` with `:before`, `:after` or `:around` as normal.


### Using with CarrierWave

Conscript supports [CarrierWave](https://github.com/carrierwaveuploader/carrierwave) uploads, but there's a couple of things you should be aware of.

First, you must ensure `register_for_draft` is called _after_ any calls to `mount_uploader`.

Then, in your uploaders where `store_dir` is defined, if you are organising file storage by model instance (e.g. `#to_param`) then you should use the new model method `uploader_store_param` to define the unique location, e.g:

    class ArticleImageUploader < ImageUploader
      def store_dir
        "public/images/articles/#{model.uploader_store_param}"
      end
    end

This will result in uploads for drafts being stored in the same location as the original instance. This is because Conscript does not want to have to worry about moving files when publishing an instance.

Conscript also overrides CarrierWave's `#destroy` callbacks to ensure that no other instance is using the same file before deleting it from the filesystem. Otherwise this can happen when you delete a draft with the same file as the original instance.

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
