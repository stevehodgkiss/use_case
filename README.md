# UseCase

Provides a convention for modelling user interactions as use case classes. A
use case class represents something a user does with your application, and is named
after it - `SignUp`, `RequestResetPasswordEmail` etc.

## Type coercion

Whitelisting and sanitizing untrusted input into an expected type before it
enters your application is a good practice. Attributes are defined and coerced
with Virtus. An attribute will either be the defined type or nil.

Take a look at the [code itself](https://github.com/stevehodgkiss/use_case/blob/master/lib/use_case.rb) for details of the API provided.

## Sign up example

Use Form objects when the input is complex or you want to bind it to a Rails
form. Attributes can also be defined on use case classes for simpler scenarios.

```ruby
class SignUpForm
  include UseCase::Params
  # this just includes Virtus configured to strictly coerce input while
  # allowing nil values:
  # include Virtus.model(strict: true, required: false)
  include ActiveModel::Validations

  attribute :username, String
  attribute :password, String
  attribute :password_confirmation, String

  # Context free validations
  validates :username, :email, :password, presence: true
  validates :password, confirmation: true

  validates :password_is_strong

  private

  def password_is_strong
    # ...
  end
end
```

The use case class interacts with the rest of your application to perform the
desired behaviour. Implement `#perform` and call `failure` fail the use case and
exit it immediately. This is similar to sinatra's `halt` in that it uses
`throw` and `catch` to terminate the method. The validation helper `validate!`
is implemented as `failure unless valid?`.

```ruby
class SignUp
  include UseCase
  include UseCase::ValidationHelpers
  include ActiveModel::Validations

  def initialize(form)
    @form = form
  end

  validate :form_is_valid

  # Contextual validations
  validate :username_is_unique

  def perform
    retry_once ActiveRecord::RecordNotUnique do
      validate!
      create_user
    end
    deliver_welcome_email
  end

  # Expose objects to the controller/other callers
  attr_reader :user

  def error_message
    errors.full_messages.to_sentence
  end

  private

  def username_is_unique
    errors.add(:username, 'is taken') if User.username_taken?(username)
  end

  def form_is_valid
    merge_errors(@form) unless @form.valid?
  end

  def create_user
    @user = User.create!(form.attributes)
  end

  def deliver_welcome_email
    UserMailer.deliver_signup_confirmation(@user)
  end

  def retry_once
    # ...
  end
end
```

## Usage inside a Rails controller

```ruby
class UsersController < ApplicationController
  before_filter :load_form, only: [:new, :create]

  def create
    use_case = SignUp.perform(@form)
    if use_case.success?
      redirect_to use_case.user
    else
      flash.now[:error] = use_case.error_message
      render :new
    end
  end

  private

  def load_form
    @form = SignUpForm.new(params)
  end
end
```

## Contributing

1. Fork it ( http://github.com/stevehodgkiss/use_case/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
