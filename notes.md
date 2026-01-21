## Details

Fixes https://github.com/rails/rails/issues/56158

The changes in https://github.com/rails/rails/pull/44127 ended up changing the transaction states so they can transition **from** `invalidated` **to** `rolledback`. That is misleading, because a `rolledback` transaction can still seen as a potentially healthy transaction, which is not true for MySQL. The following piece of code becomes misleading and can lead developers to erroneously believe they are still working with transactions when they are in fact gone:

```ruby
Account.transaction do
  Account.create!(name: "Account in top-level transaction", balance: 50)

  operation_raising_a_deadlock rescue nil # transaction is discarded with all pending changes, but the code keeps going:

  # This will write directly to the database - no valid transactions in here.
  Account.create!(name: "Last account in top-level transaction", balance: 999)

  # this will be a no-op since there's no transaction open
  raise ActiveRecord::Rollback
end

```

## Historical Context

https://github.com/rails/rails/pull/44127 was trying to fix DB connections being discarded once all the transactions in it were flagged as invalidated. From the original PR:

> PR #30922 identified that MySQL cannot handle rollbacks of savepoints once a serialization failure or deadlock has occurred. It added an :invalidated transaction state to track this and used it to skip all rollbacks.
> 
> With these transactions no longer being rolled back, the underlying connections began to be thrown away. The connection discard facility was added in #40541 to address errors during rollback.
> 
> Unfortunately, the combination of #30922 and #40541 resulted in throwing away the connection upon every deadlock and serialization failure--assuredly not what either PR intended.