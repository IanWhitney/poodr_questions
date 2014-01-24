For this implementation:

```ruby
def mirror_exists?
  count_query = Etl::MirrorQueryBuilder.new(Dars::JobQueueRun,self.receipt.id).count_sql
  cursor = ActiveRecord::Base.connection.execute(count_query)
  cursor.fetch.first.to_i != 0
end
```

I had to do this much setup

```ruby
describe "#mirror_exists?", :focus => true do
  before do
    @mirror_query_class_double = class_double("Etl::MirrorQueryBuilder").as_stubbed_const
    @ar_base_double = class_double("ActiveRecord::Base").as_stubbed_const(:transfer_nested_constants => true)

    @receipt             = instance_double("Etl::Receipt").as_null_object
    @mirror_query_double = instance_double("Etl::MirrorQueryBuilder").as_null_object
    @cursor_double       = instance_double("OCI8::Cursor").as_null_object
    @connection_double = OpenStruct.new

    receipt_id = rand(20)
    @sql_double = String.new

    @receipt.stub(:id)                                   { receipt_id }
    @cursor_double.stub(:fetch)                          { [] }
    @mirror_query_double.stub(:count_sql)                { @sql_double }
    @connection_double.stub(:execute) { @cursor_double }
    @ar_base_double.stub(:configurations) { OpenStruct.new }
    @ar_base_double.stub(:connection) { @connection_double }
    @ar_base_double.stub(:connection_id) { rand(2) }
    @ar_base_double.stub(:clear_active_connections!) { nil }

    @it = Etl::Service.new(@receipt)
  end

  it "gets the count query from MirrorQueryBuilder and the JobQueueRun table" do
    @mirror_query_class_double.should_receive(:new).with(Dars::JobQueueRun, @receipt.id) {@mirror_query_double}
    ActiveRecord::Base.stub_chain(:connection, :execute) {@cursor_double}
    @connection_double.should_receive(:execute).with(@sql_double) { @cursor_double }

    @it.mirror_exists?
  end

  it "is true if the mirror query returns a value greater than 0" do
    row_double = [rand(1..10)]
    @cursor_double.stub(:fetch) {row_double}
    @mirror_query_class_double.should_receive(:new).with(Dars::JobQueueRun, @receipt.id) {@mirror_query_double}
    ActiveRecord::Base.stub_chain(:connection, :execute) {@cursor_double}
    @connection_double.should_receive(:execute).with(@sql_double) { @cursor_double }

    expect(@it.mirror_exists?).to be_true
  end

  it "is false if the mirror query returns a value less than or equal to 0" do
    row_double = [rand(1..10) * [0,-1].sample]
    @cursor_double.stub(:fetch) {row_double}
    @mirror_query_class_double.should_receive(:new).with(Dars::JobQueueRun, @receipt.id) {@mirror_query_double}
    ActiveRecord::Base.stub_chain(:connection, :execute) {@cursor_double}
    @connection_double.should_receive(:execute).with(@sql_double) { @cursor_double }

    expect(@it.mirror_exists?).to be_false
  end
end
```

Surely there's a better way.
