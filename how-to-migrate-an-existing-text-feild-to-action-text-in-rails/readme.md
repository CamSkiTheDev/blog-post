Recently, I was assigned the task of converting some plain text to rich text in a Rails application. I know it sounds so simple, all you have to do is install Action Text, change the model in question, write a simple migration to convert the plain text to rich text and bob's your uncle, right? Not so fast.

Everything was going swimmingly until I went to write the migration. If you didn't know, Action Text doesn't store the rich text in the same database table as the model. Instead, it creates a polymorphic relationship. Action Text even creates a migration to make a new table when installed. Unfortunately, converting the text is going to be a little more complex than just refactoring our model to have a field with the type "has_rich_text". However, our model does need this so let's go ahead and add it now.

```ruby
models/post.rb

class Post < ApplicationRecord
  has_rich_text :body
end
```

Thanks to [Felicián Hoppál](https://github.com/felici), and their comment on this [issue](https://github.com/rails/rails/issues/35002#issuecomment-562311492) for a great jumping-off point and simple solution. The only problem is this migration fails when trying to roll back. The reason it fails is the "post.old_body" column doesn't exist. Not to fear, we can overcome this issue by writing separate up and down methods in our migration.

```ruby
example form github issue

class MigratePostContentToActionText < ActiveRecord::Migration[6.0]
  include ActionView::Helpers::TextHelper
  def change
    rename_column :posts, :body, :old_body
    Post.all.each do |post|
      post.update_attribute(:body, simple_format(post.old_body))
    end
    remove_column :posts, :old_body
  end
end
```

So let's create our migration and import Text Helper from Action Text.

```bash
rails g migration ConvertPostBodyToRichText
```

Now, we can import Text Helper to help convert plain text to rich text.

```ruby
20211229035004_convert_post_body_to_rich_text.rb

class ConvertPostBodyToRichText < ActiveRecord::Migration[6.1]
  include ActionView::Helpers::TextHelper
end
```

Great, so with that out of the way we can start to work on the up method of our migration. We'll start by renaming the column we want to convert to a different name. This allows for the creation/association of the relationship with the Action Text table.

```ruby
20211229035004_convert_post_body_to_rich_text.rb

class ConvertPostBodyToRichText < ActiveRecord::Migration[6.1]
  include ActionView::Helpers::TextHelper
  def up
    rename_column :posts, :body, :old_body
  end
end
```

Now, we'll loop through all of our posts and convert them to rich text with the help of the text helper from Action Text.

```ruby
20211229035004_convert_post_body_to_rich_text.rb

class ConvertPostBodyToRichText < ActiveRecord::Migration[6.1]
  include ActionView::Helpers::TextHelper
  def up
    rename_column :posts, :body, :old_body
    Post.all.each do |post|
      post.update_attribute(:body, simple_format(post.old_body))
    end
  end
end
```

Finally, we're going to delete the column we renamed since we don't need it anymore.

```ruby
20211229035004_convert_post_body_to_rich_text.rb

class ConvertPostBodyToRichText < ActiveRecord::Migration[6.1]
  include ActionView::Helpers::TextHelper
  def up
    rename_column :posts, :body, :old_body
    Post.all.each do |post|
      post.update_attribute(:body, simple_format(post.old_body))
    end
    remove_column :posts, :old_body, :text
  end
end
```

The up method is now finished and we can work on the down method. Remember, the error that occurs on rollback is an undefined method error because the old, renamed column doesn't exist anymore. So the first thing we will do in our down method is create/add that column back.

```ruby
20211229035004_convert_post_body_to_rich_text.rb

class ConvertPostBodyToRichText < ActiveRecord::Migration[6.1]
  include ActionView::Helpers::TextHelper
  def up
    rename_column :posts, :body, :old_body
    Post.all.each do |post|
      post.update_attribute(:body, simple_format(post.old_body))
    end
  end

  def down
    add_column :posts, :old_body, :text
  end
end
```

Now that we have created our new column to store our text while we roll back, let's loop through all of our posts and convert our rich text to plain text. We also have to delete the related row in the Action Text table.

```ruby
20211229035004_convert_post_body_to_rich_text.rb

class ConvertPostBodyToRichText < ActiveRecord::Migration[6.1]
  include ActionView::Helpers::TextHelper
  def up
    rename_column :posts, :body, :old_body
    Post.all.each do |post|
      post.update_attribute(:body, simple_format(post.old_body))
    end
  end

  def down
    add_column :posts, :old_body, :text
    Post.all.each do |post|
      post.update_attribute(:old_body, post.body.to_plain_text)
      post.body.delete
    end
  end
end
```

Finally, we rename the database column back to its original name.

```ruby
20211229035004_convert_post_body_to_rich_text.rb

class ConvertPostBodyToRichText < ActiveRecord::Migration[6.1]
  include ActionView::Helpers::TextHelper
  def up
    rename_column :posts, :body, :old_body
    Post.all.each do |post|
      post.update_attribute(:body, simple_format(post.old_body))
    end
  end

  def down
    add_column :posts, :old_body, :text
    Post.all.each do |post|
      post.update_attribute(:old_body, post.body.to_plain_text)
      post.body.delete
    end
    rename_column :posts, :old_body, :body
  end
end
```

Finally, we can run the migration and safely roll back if needed, with confidence to deploy to production. You should always create a database backup before running a new migration to be safe.
