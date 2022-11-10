# Syncing Rosetta with ArchivesSpace or Aleph

## Retrieve Rosetta IE Metadata

```python
def get_ie_md(ie_pid):
    wsdl = ie_wsdl
    oai_xsd = 'xmlns:schemaLocation="http://www.openarchives.org/OAI/2.0/oai_dc/ http://www.openarchives.org/OAI/2.0/oai_dc.xsd"'
    client = Client(wsdl=wsdl, settings=default_settings)
    try:
        tree_string = client.service.getMD(pdsHandle=check_handle(), PID=ie_pid)
        if oai_xsd in tree_string:
            tree_string = tree_string.replace(oai_xsd, "")
    except zeep.exceptions.Fault as e:
        return None
    tree_object = etree.fromstring(tree_string.encode("utf-8"))
    return tree_object
```

## Aleph Sync

### Metadata Crosswalk

- Look for ID in IE Tree using XPath

```python
aleph_id = tree.xpath("//dc:identifier[not(@*)]", namespaces=nsmap)
```

- If Aleph ID found, find Aleph record using Aleph API

```python
def get_marcxml_from_aleph(aleph_id):
    aleph_url = "http://baseurl:8080"
    url = r"{}/rest-dlf/record/CJH01{}&view=full".format(aleph_url, aleph_id)
    request = Request(url)
    try:
        response = urlopen(request)
        data = response.read()
        xml_tree = etree.fromstring(data)
        datafield = xml_tree.xpath('//*[local-name()="datafield"]')
        if datafield:
            record = xml_tree.xpath("//record")
            return etree.ElementTree(record[0])
    except etree.XMLSyntaxError:
        return None
```

- Crosswalk MARCXML into Rosetta DC

### Update Metadata in Rosetta

```python
aleph_dc = crosswalked_md
wsdl = ie_wsdl
client = Client(wsdl=wsdl settings=default_settings)
dc_elem_text = etree.tostring(aleph_dc, encoding="utf-8")
client.service.updateMD(
    pdsHandle=check_handle(),
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
repo = return "".join(repos[partner])
```

- Find ArchivesSpace URI from Ref ID

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
caption = "View Folder"
link = "{}/delivery/DeliveryManagerServlet?dps_pid={}".format(ros_base_url, ie_pid)
digital_object_id = "ROS_{}".format(ie_pid)
title = dc_tree.xpath("//dc:title/text()", namespaces=nsmap)[0]
row_obj = {**digital_obj_template}
row_obj["title"] = title
row_obj["file_versions"][0]["file_uri"] = link
row_obj["file_versions"][0]["caption"] = caption
row_obj["digital_object_id"] = digital_object_id
digital_obj_path = repo + "/digital_objects"
return_val = aspace.client.post(digital_obj_path, json=row_obj)
```

#### Get Archival Object JSON

```python
ref_json = aspace.client.get("{}/find_by_id/archival_objects?ref_id[]={}".format(repo, ref_id)
).json()
ao_uri = ref_json["archival_objects"][0]["ref"]
ao_json = aspace.client.get(ao_uri).json()
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
return_val = aspace.client.post(ao_uri, json=new_ao)
return_obj = return_val.json()
```

### Update Metadata in Rosetta

```python
wsdl = ie_wsdl
tree_copy = copy.deepcopy(tree)
client = Client(wsdl=wsdl settings=default_settings)
dc_elem_text = etree.tostring(aspace_dc, encoding="utf-8")
client.service.updateMD(
    pdsHandle=check_handle(),
    PID=ie_pid,
    metadata={"type": "descriptive", "subType": "dc", "content": dc_elem_text},
)
```
