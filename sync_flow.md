# Syncing Rosetta with ArchivesSpace or Aleph

- [Retrieve Rosetta IE Metadata](#retrieve-rosetta-ie-metadata)
- [Aleph Sync](#aleph-sync)
  - [Metadata Crosswalk](#metadata-crosswalk)
  - [Update Metadata in Rosetta](#update-metadata-in-rosetta)
- [ArchivesSpace Sync](#archivesspace-sync)
  - [Metadata Crosswalk](#metadata-crosswalk-1)
  - [Add Link to ArchivesSpace](#add-link-to-archivesspace)
    - [Build Digital Object](#build-digital-object)
    - [Get Archival Object JSON](#get-archival-object-json)
    - [Associate Digital Object with Archival Object](#associate-digital-object-with-archival-object)
  - [Update Metadata in Rosetta](#update-metadata-in-rosetta-1)

## Important Settings

```python
from zeep import Client, Settings
from lxml import etree
from asnake.aspace import ASpace
from urllib.request import urlopen, Request

aspace = ASpace()
default_settings = Settings(force_https=False)
```

## Retrieve Rosetta IE Metadata

```python
wsdl = ie_wsdl
oai_xsd = 'xmlns:schemaLocation="http://www.openarchives.org/OAI/2.0/oai_dc/ http://www.openarchives.org/OAI/2.0/oai_dc.xsd"'
client = Client(wsdl=wsdl, settings=default_settings)
tree_string = client.service.getMD(pdsHandle=TOKEN, PID=ie_pid)
if oai_xsd in tree_string:
    tree_string = tree_string.replace(oai_xsd, "")
tree_object = etree.fromstring(tree_string.encode("utf-8"))
return tree_object
```

## Aleph Sync

### Metadata Crosswalk

- Look for ID in IE Tree using XPath

```python
ids = tree.xpath("//dc:identifier[not(@*)]", namespaces=nsmap)
for i in ids:
    find_num = re.search("\d{9}", i.text)
    if find_num:
        aleph_id = find_num.group(0)
        return aleph_id
```

- Find Aleph record using Aleph API

```python
aleph_url = "http://baseurl:8080"
url = r"{}/rest-dlf/record/CJH01{}&view=full".format(aleph_url, aleph_id)
request = Request(url)
response = urlopen(request)
data = response.read()
xml_tree = etree.fromstring(data)
datafield = xml_tree.xpath('//*[local-name()="datafield"]')
if datafield:
    record = xml_tree.xpath("//record")
    aleph_xml = etree.ElementTree(record[0])
```

- Crosswalk MARCXML into Rosetta DC

### Update Metadata in Rosetta

```python
aleph_dc = crosswalked_md
wsdl = ie_wsdl
client = Client(wsdl=wsdl settings=default_settings)
dc_elem_text = etree.tostring(aleph_dc, encoding="utf-8")
client.service.updateMD(
    pdsHandle=TOKEN,
    PID=ie_pid,
    metadata={"type": "descriptive", "subType": "dc", "content": dc_elem_text},
)
```

## ArchivesSpace Sync

### Metadata Crosswalk

- Look for ArchivesSpace Ref ID in IE XML tree

```python
ref_id = tree.xpath('//dc:identifier[@xsi:type="dcterms:Archivesspace"]/text()', namespaces=nsmap)[0]
```

- Look for ArchivesSpace Repository in IE XML tree

```python
repos = {
    "Center for Jewish History": "repositories/2",
    "American Jewish Historical Society": "repositories/3",
    "American Sephardi Federation": "repositories/4",
    "Leo Baeck Institute": "repositories/5",
    "Yeshiva University Museum": "repositories/6",
    "YIVO Institute for Jewish Research": "repositories/7",
}
partner = tree.xpath(' //*[@id="authorativeName"]/text()', namespaces=nsmap)[0]
repo = "".join(repos[partner])
```

- Get Archival Object JSON from Ref ID

```python
ref_json = aspace.client.get(
    "{}/find_by_id/archival_objects?ref_id[]={}".format(repo, ref_id)
).json()
ao_uri = ref_json["archival_objects"][0]["ref"]
ao_json = aspace.client.get(ao_uri).json()
```

- Crosswalk ArchivesSpace JSON into Rosetta DC

### Add Link to ArchivesSpace

#### Build Digital Object

```python
row = {}
title = dc_tree.xpath("//dc:title/text()", namespaces=nsmap)[0]
caption = "View Folder"
link = "{}/delivery/DeliveryManagerServlet?dps_pid={}".format(ros_base_url, ie_pid)
digital_object_id = "ROS_{}".format(ie_pid)
row_obj = {**digital_obj_template}
row_obj["title"] = title
row_obj["file_versions"][0]["file_uri"] = link
row_obj["file_versions"][0]["caption"] = caption
row_obj["digital_object_id"] = digital_object_id
digital_obj_path = repo + "/digital_objects"
return_val = aspace.client.post(digital_obj_path, json=row_obj)
do_uri = return_val.json()["uri"]
```

#### Associate Digital Object with Archival Object

```python
row = {}
ao_instance = copy.deepcopy(instance_template)
ao_instance["digital_object"] = {"ref": do_uri}
new_ao = copy.deepcopy(archival_obj_template)
for key in ao_json.keys():
    if key in new_ao:
        new_ao[key] = ao_json[key]
not_do = [o for o in new_ao["instances"] if "digital_object" not in o.keys()]
new_ao["instances"] = [*not_do, ao_instance]
aspace.client.post(ao_uri, json=new_ao)
```

### Update Metadata in Rosetta

```python
wsdl = ie_wsdl
tree_copy = copy.deepcopy(tree)
client = Client(wsdl=wsdl settings=default_settings)
dc_elem_text = etree.tostring(aspace_dc, encoding="utf-8")
client.service.updateMD(
    pdsHandle=TOKEN,
    PID=ie_pid,
    metadata={"type": "descriptive", "subType": "dc", "content": dc_elem_text},
)
```
