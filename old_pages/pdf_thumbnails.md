## Cheatsheet (Jump to)
* **[General Overview](#general-overview)**
* **[Generating the CSV](#generating-the-csv)** with `vault/create_fileset_csv.rb`
* **[Generating Thumbnail Images](#generating-thumbnail-images)** with `vault/generic_thumbnails_script.rb`
* **[Views Where Thumbnails Appear](#views-where-thumbnails-appear)**

By default, Vault/Hyku uses the [Hydra Derivates gem](https://github.com/samvera/hydra-derivatives) and ImageMagick to generate PDF thumbnails (see [code here](https://github.com/samvera/hyrax/blob/5a9d1be16ee1a9150646384471992b03aab527a5/app/services/hyrax/file_set_derivatives_service.rb#L49)). This works fine for PDFs that contain only either text or image, but we often have PDFs that contain images and text (the latter is often produced with [OCR](https://en.wikipedia.org/wiki/Optical_character_recognition)). The automatically generated thumbnails for this kind of PDF turn out black and white with just text and no images.

Currently, the workaround for this involves manually generating thumbnails with plans to hook it into the upload process in the future. We generate one set of thumbnails per collection.

## General Overview
1. Run a script to generate a CSV with 2 columns: a) the local file path (on the server) for the PDF, b) the file set ID in Vault
2. Run another script to generate thumbnails with the name `#{first_fileset.id}-thumb.jpg` (e.g. `0c99c0d9-5ecf-4c04-a9c5-eb3a00310b1b-thumb.jpg`)
3. Place the thumbnails in a directory `apps/assets/images/thumbnails/#{Collection.title.parameterize.underscore}` (e.g. thumbnails for the Official Gazettes of British Columbia go in `apps/assets/images/thumbnails/official_gazettes_of...`). The latter part of the path gives us a directory name that corresponds to the collection title in [snake case](https://simple.wikipedia.org/wiki/Snake_case). See `apps/assets/images/thumbnails/how_to_name_new_thumbnails.txt` for a summary of all the names for directories and files generated through this process.
4. Get Vault to check if an item is a PDF and, if so, point to this new thumbnail instead of the old one. I've already changed Vault to search for the correct thumbnail. See [here](#views-where-thumbnails-appear) for the list of views I've changed. Currently the paths are hard-coded and indexed in Solr. I'm hoping to change and (re)index the paths for new and existing items in future.

## Generating the CSV
There is a script in the `home/sufia/vault` directory titled `create_fileset_csv.rb`. It has the following example code:
```ruby
require 'csv'
require 'fileutils'

CSV.open("clippings_filesets.csv", "a+") do |csv|
  csv << ["file_path", "fileset_id"]
  clippings_works.each do |work|
    work.members.each do |file_set|
      file_path = file_set.import_url
      if file_path.split(".").last == "pdf"
        fileset_id = file_set.id
        csv << [file_path, fileset_id]
      end
    end
  end
end
```
To use it, copy/paste and enter each line into the rails console with some changes:
* Replace `"clippings_filesets.csv"` with the name of the CSV to be generated
* Replace `clippings_works` with the name of an array containing all works in the collection. You can grab this through the rails console with the command `GenericWork.where(member_of_collection_ids_ssim: [Collection id])`
* Copy/paste the script into the rails console

## Generating Thumbnail Images

The thumbnails script (`generic_thumbnails_script.rb`) has the following dependencies:
* [Combine PDF](https://github.com/boazsegev/combine_pdf) gem
* [PDF to image](https://github.com/robflynn/pdftoimage) gem
* [poppler-utils](https://en.wikipedia.org/wiki/Poppler_(software)), installed via `sudo apt-get install poppler-utils` (Ubuntu/Linux) or `yum install poppler-utils` (CentOS)

Here is the full script:
```ruby
require 'combine_pdf'
require 'pdftoimage'
require 'csv'
require 'fileutils'
require 'pathname'

# For each line in the CSV
  CSV.foreach('clippings_3.csv', headers: true, header_converters: :symbol) do |row|
    # Take the first page of the pdf with that file name
    file_path = row[:file_path]
    fileset_id = row[:fileset_id]
    # byebug
    puts "#{file_path}"
    first_page = CombinePDF.load(file_path).pages[0]
    new_pdf = CombinePDF.new
    new_pdf << first_page
    # Create a new directory named with the matching work ID
    # dir = FileUtils.mkdir("thumbnails/#{fileset_id}")
    first_page_path = "clippings_3/#{File.basename(file_path).split(".")[0].gsub("&","_")}-cover.pdf" # & in a file name causes this to fail
    new_pdf.save first_page_path
    # Resize, create and save a thumbnail in that directory
    image = PDFToImage.open(first_page_path).first.resize("50%").save("clippings_3/#{fileset_id}-thumb.jpg")
    File.delete(first_page_path)
  end
```

To generate thumbnails, make the following changes:
* Replace `clippings_3.csv` with the name of the CSV generated in the [previous step](#generating-the-csv)
* Create a working directory with a name of your choice. It will contain a one-page PDF from the source PDF. The former will be deleted after the generation process. Replace **both instances** of `clippings_3` with the name of your working directory.

Now just save and run the script with `ruby generic_thumbnails_script.rb`. Once the script finishes, select all images in your working directory and move them to `app/assets/images/thumbnails/#{Collection.title.parameterize.underscore}`.

## Views Where Thumbnails Appear
I'm hoping to change and (re)index the paths for new and existing items in future.
All the following views are partials.  

In `views/catalog`
* `index_gallery`
* `index_masonry`
* `index_slideshow`
* `thumbnail_list_default`  

In `views/collections`
* `list_works`
* `show_document_list_row`   

In `views/hyrax/base`
* `member`  

In `views/hyrax/homepage`
* `featured_fields`
* `list_works`  

In `views/dashboard`
* `/collections/show_document_list_row`
* `/works/list_works`


