notification :tmux, 
  color_location: 'status-left-bg',
  default_message_format: '%s >> %s',
  display_message: true, 
  line_separator: ' > ',
  timeout: 5

guard :shell do
  watch(/^(.*)[.]md$/) do |m|
    source_file = m[0]
    dest_file = "#{m[1]}.html"
    puts "#{source_file} -> #{dest_file}"
    `grip --gfm --export #{source_file} #{dest_file}`
  end
end
