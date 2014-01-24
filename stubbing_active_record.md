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
    @ar_base_double            = class_double("ActiveRecord::Base").as_stubbed_const

    @receipt             = instance_double("Etl::Receipt")
    @mirror_query_double = instance_double("Etl::MirrorQueryBuilder")
    @cursor_double       = instance_double("OCI8::Cursor")
    @connection_double   = Object.new
    @sql_double          = String.new

    receipt_id = rand(20)

    @receipt.stub(:id)                                  { receipt_id }
    @cursor_double.stub(:fetch)                         { [] }
    @mirror_query_double.stub(:count_sql)               { @sql_double }
    @ar_base_double.stub(:connection)                   { @connection_double }
    @connection_double.stub(:execute).with(@sql_double) { @cursor_double }
    @mirror_query_class_double.stub(:new)
      .with(Dars::JobQueueRun, @receipt.id)             { @mirror_query_double}

    #magic AR stubs that you have to have
    @ar_base_double.stub(:configurations) { OpenStruct.new }
    @ar_base_double.stub(:clear_active_connections!) { nil }

    @it = Etl::Service.new(@receipt)
  end

  it "gets the count query from MirrorQueryBuilder and the JobQueueRun table" do
    @mirror_query_class_double.should_receive(:new).with(Dars::JobQueueRun, @receipt.id) {@mirror_query_double}
    @connection_double.should_receive(:execute).with(@sql_double) { @cursor_double }

    @it.mirror_exists?
  end

  it "is true if the mirror query returns a value greater than 0" do
    row_double = [rand(1..10)]
    @cursor_double.stub(:fetch) {row_double}

    expect(@it.mirror_exists?).to be_true
  end

  it "is false if the mirror query returns a value less than or equal to 0" do
    [0,-1].each do |mult|
      row_double = [rand(1..10) * mult]
      @cursor_double.stub(:fetch) {row_double}

      expect(@it.mirror_exists?).to be_false, "A count of #{row_double.first} should make mirror_exists? false"
    end
  end
end
```

Surely there's a better way.
