#!/usr/bin/ruby
#-*-mode:ruby-*-
# Time-stamp: <2010-11-07 12:03:45 (djcb)> 

# spuug: program to generate Gtk+ and GObject boilerplate in C
#
# Copyright (C) 2006-2009 Dirk-Jan C. Binnema <djcb@djcbsoftware.nl>
#
# spuug is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 3 of the License, or
# (at your option) any later version.

# spuug is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.

require 'getoptlong'

class GNamer 
  attr_reader :basename,:namespace, :classname, :parent_classname

  def initialize(classname,namespace,parent_classname)

    @classname,@parent_classname,@namespace = classname,parent_classname,namespace
    if @namespace
      if @classname.index(@namespace) == 0 
        @basename = @classname.slice(@namespace.length,@classname.length)
      else
        raise "classname must start with namespace"
      end
    elsif @classname =~ /^(Gtk|Gnome|Egg|Hildon)/
      @namespace = $1
      @basename = @classname.slice(@namespace.length,@classname.length)
    else
      raise "must provide a namespace"
    end
  end

  def file_name (ext)         "#{us_classname.gsub(/_/,'-')}.#{ext}" end
  def us_basename()           underscorify(@basename).downcase end
  def us_namespace()          underscorify(@namespace).downcase end
  def us_classname()          us_namespace + "_" + us_basename end
  def test_name()             "test-#{us_classname.gsub(/_/,'-')}" end    

  def parent_cast_macro()
    if @parent_classname =~ /^GObject/
      "G_OBJECT"
    else
      underscorify(@parent_classname).upcase 
    end  
  end

  def parent_type_macro     
    if @parent_classname == 'GObject'
      "G_TYPE_OBJECT"
    elsif @parent_classname =~ /^Gtk(.*)/
      "GTK_TYPE_" + underscorify($1).upcase
    elsif @parent_classname == "GTypeInterface"
      "G_TYPE_INTERFACE"
    else
      parts = split_name_parts(@parent_classname)
      if (parts.length == 1)
        "TYPE_" + parts[0].upcase
      else
        parts = parts.slice(0) + "Type" + parts.slice(1,parts.length-1).to_s
        underscorify(parts).upcase
      end
    end
  end
  
  def parent_include
    if @parent_classname =~ /^(Gtk)/
      "<gtk/gtk.h>"
    elsif @parent_classname =~ /^(GObject|GTypeInterface)/
      "<glib-object.h>"
    elsif @parent_classname =~ /^(Gnome)/
      "<libgnome/#{underscorify(@parent_classname).downcase.gsub(/_/,'-')}.h>"
    else
      "\"#{underscorify(@parent_classname).downcase.gsub(/_/,'-')}.h\"" 
    end
  end

# private:
  
  def underscorify(str)
    result = ""
    split_name_parts(str).each {|part|
      result += if result.empty? then part else "_" + part end
    }
    return result
  end
  
  def islower(c)
    if c.empty? then false else (c[0,1].downcase == c[0,1]) end
  end
  
  # FooBarCuux => (Foo,Bar,Cuux)
  def split_name_parts(name)  
    parts = Array.new
    part = name[0,1]
    
    for i in (1..name.length)
      if name[i,1] == "_" then
        parts.push(part)
        part = ""
        i += 1
        next
      end
      if islower(name[i-1,1]) and not islower(name[i,1]) then
        parts.push(part)
        part = ""
      end
      part += name[i,1]
    end
    if not part.empty? then parts.push(part) end
    parts
  end 
  
  private :split_name_parts,:islower,:underscorify
end


class GObjectBuilder
  
  def initialize(gnamer) # a GNamer instance 
    @gnamer           = gnamer 
    @type_macro       = "#{@gnamer.us_namespace}_TYPE_#{@gnamer.us_basename}".upcase
    @cast_macro       = "#{@gnamer.us_namespace}_#{@gnamer.us_basename}".upcase
    @include_guard    = "__#{@cast_macro}_H__" 
    @is_macro         = "#{@gnamer.us_namespace}_IS_#{@gnamer.us_basename}".upcase
    @class_is_macro   = "#{@is_macro}_CLASS"  
    @class_cast_macro = "#{@cast_macro}_CLASS"
    @class_is_macro   = "#{@is_macro}_CLASS"
    @class_get_macro  = "#{@cast_macro}_GET_CLASS" 
    @longest          = "#{@class_is_macro}(klass)".length
    @private_struct   = "#{@gnamer.classname}Private"
  end

  def build_iface_dot_h
    dot_h = 
      commentnl(@gnamer.file_name("h")) +
      commentnl("insert (c)/licensing information)") + "\n" +
      "#ifndef #{@include_guard}\n" +
      "#define #{@include_guard}\n\n" +
           
      if @gnamer.parent_classname != "GTypeInterface" 
      then
        "#include #{@gnamer.parent_include}\n"
      else 
        ""
      end +

      commentnl("other include files") + "\n" +
      "G_BEGIN_DECLS\n\n" +
      commentnl("convenience macros") +
      
      hash_define(@type_macro, 
                  "(#{@gnamer.us_classname}_get_type())",
                  @longest) + "\n" +
      hash_define("#{@cast_macro}(obj)",
                  "(G_TYPE_CHECK_INSTANCE_CAST((obj)," + 
                  @type_macro + "," + @gnamer.classname + "))", 
                  @longest) + "\n" +
      hash_define("#{@is_macro}(obj)",
                  "(G_TYPE_CHECK_INSTANCE_TYPE((obj)," +
                  @type_macro + "))",
                  @longest) + "\n" +
      hash_define("#{@class_get_macro}(inst)", 
                  "(G_TYPE_INSTANCE_GET_INTERFACE((inst)," +
                  @type_macro + ",#{@gnamer.classname}Class))",
                  @longest) + "\n\n" +
      
      "typedef struct _#{@gnamer.classname}      #{@gnamer.classname};\n" + 
      "typedef struct _#{@gnamer.classname}Class #{@gnamer.classname}Class;\n\n" + 
      
      # parent class
      "struct _#{@gnamer.classname}Class {\n" +
      "\t#{@gnamer.parent_classname} parent;\n\n" +
      "\t" + commentnl("the 'vtable': declare function pointers here, e.g.:") +
      "\t" + commentnl("gchar* (*do_something_func) (" + 
                       @gnamer.classname + "* self, gint some_value);") +
      "};\n\n" +
      
      # main functions
      "GType        #{@gnamer.us_classname}_get_type    (void) G_GNUC_CONST;\n\n" +

      commentnl("declare interface functions, corresponding to vtable above, e.g.:") +
      commentnl("\tgchar*       " + @gnamer.us_classname + 
                "_do_something (#{@gnamer.classname} *self, gint some_value);") + "\n\n" + 
    
      "G_END_DECLS\n\n" +
      "#endif " + comment(@include_guard) + "\n\n" 
 
      dot_h
  end

  def build_iface_dot_c    
    max_func_len = "#{@gnamer.us_classname}_class_init".length
    max_arg_len  = "(#{@gnamer.classname}Class *klass);".length
    
    dot_c = 
      commentnl(@gnamer.file_name("c")) +
      commentnl("insert (c)/licensing information)") + "\n" +
      "#include \"#{@gnamer.file_name("h")}\"\n\n" +
      
      "static void #{@gnamer.us_classname}_base_init (gpointer g_class);\n\n" + 

      commentnl("'implement' our interface functions, e.g.:") +
      "/*\n" + 
      "gchar*\n" + 
      "#{@gnamer.us_classname}_do_something (#{@gnamer.classname} *self, gint some_value)\n" +
      "{\n\treturn #{@class_get_macro}(self)->\n" + 
      "\t\tdo_something_func (self, some_value);\n}\n" +
      "*/\n\n" +

      "static void\n#{@gnamer.us_classname}_base_init (gpointer g_class)\n" +
      "{\n" +
      "\tstatic gboolean initialized = FALSE;\n" +
      "\tif (!initialized) {\n" +
      "\t" + commentnl("create interface signals here") +
      "\t\tinitialized = TRUE;\n" +
      "\t}\n}\n" +
      
      commentnl("see G_DEFINE_TYPE_WITH_CODE if you need more control") +
      "G_DEFINE_TYPE (#{@gnamer.classname}, #{@gnamer.us_classname}, " +
      "#{@gnamer.parent_type_macro});\n\n"
  end

  
  def build_obj_dot_h   
    dot_h = 
      commentnl(@gnamer.file_name("h")) +
      commentnl("insert (c)/licensing information)") + "\n" +
      "#ifndef #{@include_guard}\n" +
      "#define #{@include_guard}\n\n" +
      "#include #{@gnamer.parent_include}\n" +
      commentnl("other include files") + "\n" +
      "G_BEGIN_DECLS\n\n" +
      commentnl("convenience macros") +
      
      hash_define(@type_macro, 
                  "(#{@gnamer.us_classname}_get_type())",
                  @longest) + "\n" +
      hash_define("#{@cast_macro}(obj)",
                  "(G_TYPE_CHECK_INSTANCE_CAST((obj)," + 
                  @type_macro + "," + @gnamer.classname + "))", 
                  @longest) + "\n" +
      hash_define("#{@class_cast_macro}(klass)",
                  "(G_TYPE_CHECK_CLASS_CAST((klass)," + 
                  @type_macro + "," + @gnamer.classname + "Class))",
                  @longest) + "\n" +
      hash_define("#{@is_macro}(obj)",
                  "(G_TYPE_CHECK_INSTANCE_TYPE((obj)," +
                  @type_macro + "))",
                  @longest) + "\n" +
      hash_define("#{@class_is_macro}(klass)",
                  "(G_TYPE_CHECK_CLASS_TYPE((klass)," +
                  @type_macro  + "))",
                  @longest) + "\n" +
      hash_define("#{@class_get_macro}(obj)", 
                  "(G_TYPE_INSTANCE_GET_CLASS((obj)," +
                  @type_macro + ",#{@gnamer.classname}Class))",
                  @longest) + "\n\n" +
      
      "typedef struct _#{@gnamer.classname}      #{@gnamer.classname};\n" + 
      "typedef struct _#{@gnamer.classname}Class #{@gnamer.classname}Class;\n" + 
      "typedef struct _#{@private_struct}         #{@private_struct};\n\n" +

      # class 
      "struct _#{@gnamer.classname} {\n" +
      "\t #{@gnamer.parent_classname} parent;\n" +
      "\t" + commentnl("insert public members, if any") + "\n" +
      "\t" + commentnl("private") +
      "\t" + "#{@gnamer.classname}Private *_priv;\n" +
      "};\n\n" +
      
      # parent class
      "struct _#{@gnamer.classname}Class {\n" +
      "\t#{@gnamer.parent_classname}Class parent_class;\n" +
      "\t" + commentnl("insert signal callback declarations, e.g.") +
      "\t" + commentnl("void (* my_event) (" + @gnamer.classname + "* obj);") +
      "};\n\n" +
      
      # main functions
      commentnl("member functions") +
      "GType        #{@gnamer.us_classname}_get_type    (void) G_GNUC_CONST;\n\n" +
      commentnl("parameter-less _new function (constructor)") +

      commentnl("if this is a kind of GtkWidget, it should probably return at GtkWidget*") +
      if (@gnamer.namespace =~ /Gtk|Gnome|Egg|Hildon/ or 
          @gnamer.parent_classname =~ /Gtk|Gnome|Egg|Hildon/)
        then "GtkWidget*   " else   "#{@gnamer.classname}*    " end +

      @gnamer.us_classname + "_new         (void);\n\n"+
      commentnl("fill in other public functions, e.g.:") +
      commentnl("\tvoid       " + @gnamer.us_classname + 
                "_do_something (#{@gnamer.classname} *self, const gchar* param);") +
      commentnl("\tgboolean   " + @gnamer.us_classname + 
                "_has_foo      (#{@gnamer.classname} *self, gint value);") + "\n\n" +
    
      "G_END_DECLS\n\n" +
      "#endif " + comment(@include_guard) + "\n\n" 
    return dot_h
  end
  
  def build_obj_dot_c

    get_private_func     = "#{@gnamer.us_classname.upcase}_GET_PRIVATE"
    get_private_func_def = "#define #{get_private_func}(o)"
    
    max_func_len = "#{@gnamer.us_classname}_class_init".length
    max_arg_len  = "(#{@gnamer.classname}Class *klass);".length

    dot_c = 
      commentnl(@gnamer.file_name("c")) + "\n" +
      commentnl("insert (c)/licensing information)") + "\n" +
      "#include \"#{@gnamer.file_name("h")}\"\n" +
      
      commentnl("include other impl specific header files") + "\n" +
      
      commentnl("'private'/'protected' functions") +
      "static void #{@gnamer.us_classname}_class_init (#{@gnamer.classname}Class *klass);\n" +
      "static void #{@gnamer.us_classname}_init       (#{@gnamer.classname} *obj);\n" +
      "static void #{@gnamer.us_classname}_finalize   (GObject *obj);\n\n" + 
      
      commentnl("list my signals ") +
      "enum {\n" +
      "\t" + commentnl("MY_SIGNAL_1,") +
      "\t" + commentnl("MY_SIGNAL_2,") +
      "\tLAST_SIGNAL\n};\n\n" +
      
      "struct _#{@private_struct} {\n" +
      "\t" + commentnl("my private members go here, e.g.") +
      "\t" + commentnl("gboolean _frobnicate_mode;") +
      "};\n" +
      align2(get_private_func_def, "(G_TYPE_INSTANCE_GET_PRIVATE((o), \\",
             get_private_func_def.length + 4) + "\n" +
      align2("", " #{@type_macro}, \\",
             get_private_func_def.length + 4) + "\n" +
      align2("", " #{@private_struct}))",
             get_private_func_def.length + 4) + "\n" +
      commentnl("globals") +
      "static #{@gnamer.parent_classname}Class *parent_class = NULL;\n\n" +
      commentnl("uncomment the following if you have defined any signals") + 
      commentnl("static guint signals[LAST_SIGNAL] = {0};") + "\n" +
      
      "G_DEFINE_TYPE (#{@gnamer.classname}, #{@gnamer.us_classname}, " +
                      "#{@gnamer.parent_type_macro});\n\n" +

      "static void\n#{@gnamer.us_classname}_class_init (#{@gnamer.classname}Class *klass)\n" +
      "{\n" +
      "\tGObjectClass *gobject_class;\n" +
      "\tgobject_class = (GObjectClass*) klass;\n\n" +
      "\tparent_class            = g_type_class_peek_parent (klass);\n" +
      "\tgobject_class->finalize = #{@gnamer.us_classname}_finalize;\n\n" +
      "\tg_type_class_add_private (gobject_class, sizeof(#{@private_struct}));\n\n" +
      "\t" + commentnl("signal definitions go here, e.g.:") +
      commentnl("\tsignals[MY_SIGNAL_1] =") +
      commentnl("\t\tg_signal_new (\"my_signal_1\",....);") + 
      commentnl("\tsignals[MY_SIGNAL_2] =") +
      commentnl("\t\tg_signal_new (\"my_signal_2\",....);") +
      commentnl("\tetc.") +
      "}\n\n" +
      
      "static void\n#{@gnamer.us_classname}_init (#{@gnamer.classname} *obj)\n" +
      "{\n" +
      "\tobj->_priv = #{get_private_func}(obj);\n\n" +
      commentnl("to init any of the private data, do e.g:") + 
      commentnl("\tobj->_priv->_frobnicate_mode = FALSE;") +
      "}\n\n" +
      
      "static void\n#{@gnamer.us_classname}_finalize (GObject *obj)\n" +
      "{\n" +
      commentnl("\tfree/unref instance resources here") +
      "\tG_OBJECT_CLASS(parent_class)->finalize (obj);\n" +
      "}\n\n" +
      
      if (@gnamer.namespace =~ /Gtk|Gnome|Egg|Hildon/ or 
          @gnamer.parent_classname =~ /Gtk|Gnome|Egg|Hildon/)
      then "GtkWidget*" else "#{@gnamer.classname}*" end +
      
      "\n#{@gnamer.us_classname}_new (void)\n" +
      "{\n" +
      "\treturn " + 
      if (@gnamer.namespace =~ /Gtk|Gnome|Egg|Hildon/ or 
          @gnamer.parent_classname =~ /Gtk|Gnome|Egg|Hildon/)
      then "GTK_WIDGET" else "#{@cast_macro}" end +
      "(g_object_new(#{@type_macro}, NULL));\n" +
      "}\n\n" +
      commentnl("following: other function implementations") +
      commentnl("such as #{@gnamer.us_classname}_do_something, or " +
                "#{@gnamer.us_classname}_has_foo") + "\n"
  end
  
  
def build_test_obj_dot_c
  commentnl("unit test for the #{@gnamer.classname} implementation") + "\n" +
    
    if @gnamer.parent_classname =~ /^Gtk/
      "#include <gtk/gtk.h>\n"
    else "" end +

    "#include \"#{@gnamer.file_name("h")}\"\n\n" +
    "int\nmain (int argc, char* argv[])\n{\n" +
    
    if @gnamer.parent_classname =~ /^Gtk/
      "\tGtkWidget *obj;\n\n" 
    elsif @gnamer.parent_classname =~ /^GObject/
      "\tGObject *obj;\n\n" 
    else
      "\t#{@gnamer.classname} *obj;\n\n" 
    end +

    if @gnamer.parent_classname =~ /^(Gtk|Gnome|Egg)/
      "\tgtk_init (&argc, &argv);\n"
    elsif @gnamer.parent_classname =~ /^(GObject)/
      "\tg_type_init ();\n"
    else
      commentnl("\tput init code here, e.g.") +
        commentnl("\tgtk_init for gtk-derived, or g_type for gobject-derived")
    end +
    
    "\n\tobj = #{@gnamer.us_classname}_new ();\n" +
    commentnl("do something interesting with our brand new object") + "\n" +

    if @gnamer.parent_classname =~ /^(Gtk|Gnome|Egg)/
      "\tgtk_main ();\n"
    else "" end +    
    "\treturn 0;\n}\n"
end

def build_test_obj_makefile

  pkg_config_mod=""
  if @gnamer.parent_classname =~ /^(Gtk)/
    pkg_config_mod='gtk+-2.0'
  elsif @gnamer.parent_classname =~ /^(GObject)/
    pkg_config_mod='gobject-2.0'
  elsif @gnamer.parent_classname =~ /^(Gnome)/
    pkg_config_mod='libgnome-2.0 libgnomeui-2.0'
  end

  "# Makefile for testing #{@gnamer.classname}\n\n" +
    "#{@gnamer.test_name}: #{@gnamer.file_name("o")} #{@gnamer.test_name}.o\n" + 
    "\t$(CC) -o #{@gnamer.test_name} #{@gnamer.test_name}.o #{@gnamer.file_name("o")} " + 
    "`pkg-config --libs #{pkg_config_mod}`\n\n" +
    
    "#{@gnamer.file_name("o")}: #{@gnamer.file_name("c")} #{@gnamer.file_name("h")}\n" +
    "\t$(CC) -c #{@gnamer.file_name("c")} "+
    "-g -Wall `pkg-config --cflags #{pkg_config_mod}`\n\n" +
    
    "#{@gnamer.test_name}.o: #{@gnamer.test_name}.c #{@gnamer.file_name("h")}\n" +
    "\t$(CC) -c #{@gnamer.test_name}.c "+
    "-g -Wall `pkg-config --cflags #{pkg_config_mod}`\n\n" +


    "clean:\n" +
    "\trm -f *.o *~ core\n"
end



# private
def comment(str)   "/* #{str} */" end
  def commentnl(str)  comment(str) + "\n"  end

  def hash_define(name,value,longest)
    align2("#define " + name,value, longest + "#define ".length)
  end

  def align2(str1,str2,longest1)
    str1 + (" " * ((longest1 + 2) - (str1.length))) + str2 
  end

  def align3(str1,str2,str3,longest1,longest2)
    align2(str1,align2(str2,str3,longest2),longest1)
  end
  
  private :comment,:commentnl,:hash_define,:align2
end

#
# main
#
def usage
  $stderr.print "spuug version 0.4\n" +
    "Copyright (c) 2006-2010 Dirk-Jan C. Binnema <djcb@djcbsoftware.nl>.\n" +
    "spuug is free software covered by the GNU GPL v3+\n\n" +
    "spuug is a script to generate GObject boilerplate code\n" +
    "usage: spuug [OPTIONS]\n" +
    "where options are:\n" +
    "\t--class=<classname>,-c         : the classname (e.g. MyFooBar)\n" +                   
    "\t--interface=<interface>,-i     : the interface name (e.g. MyFooBarIFace)\n" +
    "\t--parent=<parent-classname>,-p : the parent classname (e.g. Bar)\n" +
    "\t--namespace=<namespace>,-n     : the namespace (e.g. My)\n" +
    "\t--test,-t                      : generate test code as well\n" +
    "\t--force,-f                     : overwrite existing files\n" +
    "\t--help,-h                      : show this help text\n\n" +
    "Example:\n" +
    "\t$ spuug --class=FunkyFooBar --namespace=Funky --parent=GtkWidget\n" +
    "will generate funky-foobar.c and funky-foobar.h with the boilerplate code\n\n" +
    "and\n" +
    "\t$ spuug --class=CuteThing --namespace=Cute --parent=GObject --test\n" +
    "will generate cute-thing.c and cute-thing.h with the boilerplate code,\n" +
    "and test-cute-thing.c and Makefile for testing\n\n"
    
end
    

# parse the options
opts = GetoptLong.new(
   ["--class",     "-c", GetoptLong::REQUIRED_ARGUMENT ],                   
   ["--interface", "-i", GetoptLong::REQUIRED_ARGUMENT ],
   ["--parent",    "-p", GetoptLong::REQUIRED_ARGUMENT ],
   ["--namespace", "-n", GetoptLong::REQUIRED_ARGUMENT ],
   ["--force",     "-f", GetoptLong::NO_ARGUMENT],
   ["--test",      "-t", GetoptLong::NO_ARGUMENT],                   
   ["--help",      "-h", GetoptLong::NO_ARGUMENT]
)

classname,interface,parent_classname,namespace,force,build_test=nil
begin
  opts.each do |opt, arg|
    case opt
    when "--help"
      usage()
      exit 0
    when "--class"
      classname = arg
    when "--interface" 
      interface = arg
    when "--namespace"
      namespace = arg
    when "--force"
      force=true
    when "--test"
      build_test=true
    when "--parent"
      parent_classname = arg
    end
  end
rescue
  usage()
  exit 1
end

#
# check the parameters
#

if not interface and not classname
  usage()
  $stderr.print "error: please choose exactly one of --interface= and --class=\n"
  exit 1
end

if interface and classname 
  usage()
  $stderr.print "error: please choose either --interface or --class, not both\n"
  exit 1
end


if not parent_classname
  $stderr.print "warning: parent class not provided ('--parent=<parent-classname>')\n"
  if classname 
    $stderr.print "warning: assuming --parent=GObject\n"
    parent_classname="GObject"
  elsif interface
    $stderr.print "warning: assuming --parent=GTypeInterface\n"
    parent_classname="GTypeInterface"
  end
end

if not namespace
  usage()
  $stderr.print "error: please specify a namespace '--namespace=<namespace>'\n"
  exit 1
end

if classname
  if classname.index(namespace) != 0
    classname = namespace + classname
    $stderr.print "warning: classname '#{classname}' does " + 
      "not start with namespace '#{namespace}\n"
    $stderr.print "warning: prepending it myself => '#{classname}'\n"
  end
    
  gnamer   = GNamer.new(classname,namespace,parent_classname)

elsif interface
  if interface.index(namespace) != 0
    interface = namespace + interface
    $stderr.print "warning: interface does not start with namespace\n"
    $stderr.print "warning: prepending => '#{classname}'\n"
  end
  # interface must end in IFace or Interface
  # ruby strings could use 'ends_with' I guess...
  if interface.downcase.index("iface", interface.length - "iface".length) == nil and 
      interface.downcase.index("interface", interface.length - "interface".length) == nil
    interface += "IFace"
    $stderr.print "warning: interfaces must end in IFace or Interface\n"
    $stderr.print "warning: appending 'IFace' == '#{interface}'\n"
  end  

  gnamer   = GNamer.new(interface,namespace,parent_classname)

end


gbuilder = GObjectBuilder.new(gnamer)

dot_c=gnamer.file_name("c")
dot_h=gnamer.file_name("h")

if (File.exists?(dot_c) or File.exists?(dot_c)) and not force
  $stderr.print "error: #{dot_c} and/or #{dot_h} already exists." + 
    "Use --force to overwrite them\n"
  exit 1
end

if build_test  
  test_dot_c="#{gnamer.test_name}.c"
  makefile  ="Makefile"

  if (File.exists?(test_dot_c) or File.exists?(makefile)) and not force
    $stderr.print "error: #{test_dot_c} and/or #{makefile} already exists." + 
      "Use --force to overwrite them\n"
    exit 1
  end

  File.new(test_dot_c,"w").write(gbuilder.build_test_obj_dot_c)
  File.new(makefile,"w").write(gbuilder.build_test_obj_makefile)

end  

if classname then
  File.new(dot_c,"w").write(gbuilder.build_obj_dot_c)
  File.new(dot_h,"w").write(gbuilder.build_obj_dot_h)
elsif interface then
  File.new(dot_c,"w").write(gbuilder.build_iface_dot_c)
  File.new(dot_h,"w").write(gbuilder.build_iface_dot_h)

end
  
# the end
