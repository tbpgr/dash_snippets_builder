require 'sqlite3'
require 'toml'
require 'dotenv'
require 'logger'
require 'fileutils'
Dotenv.load

namespace :dash do
  TOML_PATH = './snippets'
  DUMP_PATH = './snippets/dump'
  SNIPPET = 'snippet'
  module Fields
    TITLE = 'title'
    BODY = 'body'
    SYNTAX = 'syntax'
    TAG = 'tag'
  end

  module OutputRecordsIndex
    TITLE = 0
    BODY = 1
    SYNTAX = 2
    TAG = 3
  end

  def build_snippets
    initialize_log
    @logger.debug('start build snippets')
    load_snippets
    connect_sqlite
    delete_tables
    @snippets.each.with_index(0) do |snippet, _i|
      records = read_snippets_by_title(snippet[Fields::TITLE])
      record_count = get_record_count(records)
      create_snippet(snippet)
    end
    @logger.debug('finish build snippets')
  end

  def dump_snippets
    initialize_log
    @logger.debug('start dump snippets')
    connect_sqlite
    snippets = load_db_snippets
    @logger.debug("\tsnippet count = #{snippets.size}")
    snippets.each do |snippet|
      outer = {}
      inner = {}
      inner[:title] = snippet[OutputRecordsIndex::TITLE]
      inner[:body] = snippet[OutputRecordsIndex::BODY]
      inner[:syntax] = snippet[OutputRecordsIndex::SYNTAX]
      inner[:tag] = snippet[OutputRecordsIndex::TAG]
      outer[:snippet] = inner
      FileUtils.mkdir_p(DUMP_PATH)
      filename = normalize_filename(inner[:title])
      File.open("#{DUMP_PATH}/#{filename}.toml", "w") {|e|e.puts TOML.dump(outer) }
      @logger.debug("\t complete output #{DUMP_PATH}/#{filename}.toml ")
    end
    @logger.debug('finish dump snippets')
  end
  private

  def initialize_log
    @logger = Logger.new(STDOUT)
    level = ENV['LOG_LEVEL']
    @logger.level = Object.const_get("Logger::#{level}")
    @logger.formatter =-> (severity, datetime, progname, message) {
      "#{datetime.strftime('%Y/%m/%d %H:%M:%S')} - #{level} - #{message}\n"
    }
  end

  def load_snippets
    @snippets = Dir.glob("#{TOML_PATH}/**/*.toml").each_with_object([]) do |e, memo|
      memo << TOML.load_file(e)[SNIPPET]
    end
  end

  def delete_tables
    @logger.debug("\tstart delete snippets")
    @db = connect_sqlite
    delete_table('snippets')
    delete_table('tagsIndex')
    delete_table('tags')
    @logger.debug("\tfinish delete snippets")
  end

  def delete_table(table_name)
    @db.execute("delete from #{table_name}")
    @logger.debug("\t\tsuccess delete #{table_name}")
  end

  def connect_sqlite
    @db = SQLite3::Database.new(ENV['DASH_SNIPPET_PATH'])
  end

  def read_snippets_by_title(title)
    @db.execute('select count(*) from snippets where title = ?', title)
  end

  def get_record_count(records)
    records.first.first
  end

  def create_snippet(snippet)
    @logger.debug("\tcreate #{snippet[Fields::TITLE]}")
    insert_tags_index(snippet)
    create_tag_if_not_exist(snippet[Fields::TAG])
    insert_snippet(snippet)
  end

  def insert_tags_index(snippet)
    @logger.debug("\t\tinsert tagsIndex for title:#{snippet[Fields::TITLE]} tag: #{snippet[Fields::TAG]}")
    if tag_exist?(snippet[Fields::TAG])
      @db.execute('insert into tagsIndex values ((select tid from tags where tag = ?), ?)', snippet[Fields::TAG], next_sid)
    else
      @db.execute('insert into tagsIndex values (?, ?)', next_tid, next_sid)
    end
  end

  def create_tag_if_not_exist(tag)
    return if tag_exist?(tag)
    @logger.debug("\t\tcreate tag #{tag}")
    @db.execute('insert into tags values ((select max(tid) from tagsIndex), ?)', tag)
  end

  def next_sid
    next_id('tagsIndex', 'sid')
  end

  def next_tid
    next_id('tags', 'tid')
  end

  def next_id(table, column)
    count = @db.execute("select count(*) from #{table}").first.first
    return 1 if count == 0
    @db.execute("select max(#{column}) from #{table}").first.first + 1
  end

  def tag_exist?(tag)
    tag_count = @db.execute('select count(*) from tags where tag = ?', tag).first.first
    tag_count == 1
  end

  def insert_snippet(snippet)
    @logger.debug("\t\tcreate snippet #{snippet[Fields::TITLE]}")
    @db.execute('insert into snippets values((select max(sid) from tagsIndex),?,?,?,?)',
                snippet[Fields::TITLE],
                snippet[Fields::BODY],
                snippet[Fields::SYNTAX],
                0
               )
  end

  def load_db_snippets
    sql =<<-SQL
select s.title, s.body, s.syntax, t.tag
from snippets s
    inner join tagsIndex ti on s.sid == ti.sid
    inner join tags t on ti.tid = t.tid
    SQL
    @db.execute(sql)
  end

  def normalize_filename(filename)
    filename.gsub(/([^a-z|A-Z|0-9|\-_])/, '')
  end

  desc 'build snippets'
  task :build do
    build_snippets
  end

  desc 'dump snippets'
  task :dump do
    dump_snippets
  end
end
