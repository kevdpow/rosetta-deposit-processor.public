# Syncing Rosetta with ArchivesSpace or Aleph

## High level code review

### 1) Retrieve Rosetta IE Metadata

- Generate list of IEs from CSV, IE List or SIP
- For each IE in the list, get IE Tree metadata

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

### Aleph Sync

#### Metadata Crosswalk

- Look for IDs in IE Tree using XPath

```python
ids = tree.xpath("//dc:identifier[not(@*)]", namespaces=nsmap)
```

- Iterate through IDs looking for Aleph numbers
- If Aleph ID found, find Aleph record using Aleph API

```python
def get_marcxml_from_aleph(aleph_id):
    url = r"http://opac.cjh.org:1891/rest-dlf/record/CJH01{}&view=full".format(aleph_id)
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

```python
def parseMARCobj(marc_obj):
    xsl = migration_xslt
    transform = etree.XSLT(xsl)
    record = marc_obj.xpath('//*[local-name()="record"]')
    if record:
        record[0].attrib["xmlns"] = "http://www.loc.gov/MARC21/slim"
        record[0].attrib[
            "{http://www.w3.org/2001/XMLSchema-instance}schemaLocation"
        ] = "http://www.loc.gov/MARC21/slim http://www.loc.gov/standards/marcxml/schema/MARC21slim.xsd"
    marc_obj = etree.fromstring(etree.tostring(marc_obj))
    dcMD_result = transform(marc_obj)
    if dcMD_result:
        dcMD_result_string = etree.tostring(dcMD_result)
        dcMD = etree.fromstring(dcMD_result_string)
        if dcMD != None:
            return dcMD
```

- Add crosswalked metadata information to `ie_dict`

```python
ie_dict[ie_pid] = {
    "tree": etree.tostring(tree_copy, encoding="utf-8"),
    "aleph_id": aleph_id,
    "aleph_dc": aleph_dc,
    "caption": caption,
}
```

#### Update Metadata in Rosetta

```python
wsdl = ie_wsdl
tree_copy = copy.deepcopy(tree)
client = Client(wsdl=wsdl settings=default_settings)
dc_elem_text = etree.tostring(aleph_dc, encoding="utf-8")
client.service.updateMD(
    pdsHandle=check_handle(),
    PID=ie_pid,
    metadata={"type": "descriptive", "subType": "dc", "content": dc_elem_text},
)
```

### ArchivesSpace Sync

#### Metadata Crosswalk

- Look for ArchivesSpace Ref ID in IE XML tree

```python
ref_id_list = tree_copy.xpath('//dc:identifier[@xsi:type="dcterms:Archivesspace"]/text()', namespaces=nsmap
```

- Look for ArchivesSpace Repository in IE XML tree

```python
partner = tree.xpath(' //*[@id="authorativeName"]/text()', namespaces=nsmap)[0]
```

- Find ArchivesSpace URI from Ref ID

```python
def get_aspace_uri_from_ref_id(ref_id, repo=None):
    def make_aspace_calls(i):
        i_str = str(i)
        ref_json_no_prefix = aspace.client.get(
            "{}/find_by_id/archival_objects?ref_id[]={}".format(i_str, ref_id)
        ).json()
        ref_json_prefix = aspace.client.get(
            "{}/find_by_id/archival_objects?ref_id[]=aspace_{}".format(i_str, ref_id)
        ).json()
        searches = [ref_json_no_prefix, ref_json_prefix]
        find = [f for f in searches if f["archival_objects"]]
        if find:
            ref_json = find[0]
            ao_uri = ref_json["archival_objects"][0]["ref"]
            ao_json = aspace.client.get(ao_uri).json()
            if "error" not in ao_json:
                return ao_json
        else:
            return None

    if repo:
        return make_aspace_calls(repo)
    else:
        for i in range(2, 8):
            ao_json = make_aspace_calls(i)
            if ao_json == None:
                continue
            else:
                return ao_json
```

- Crosswalk ArchivesSpace JSON into Rosetta DC

```python
def dc_from_aspace_json(aspace_uri):
    fields = []
    nsmap = {
        "dc": "http://purl.org/dc/elements/1.1/",
        "xsi": "http://www.w3.org/2001/XMLSchema-instance",
        "dcterms": "http://purl.org/dc/terms/",
        "dnx": "http://www.exlibrisgroup.com/dps/dnx",
        "mods": "http://www.loc.gov/mods/v3",
        "oai_dc": "http://www.openarchives.org/OAI/2.0/oai_dc/",
    }
    if "archival_objects" in aspace_uri:
        ao_json = aspace.client.get(aspace_uri).json()
    else:
        ref_id = posixpath.basename(aspace_uri)
        repo = posixpath.dirname(aspace_uri)
        uri_request = aspace.client.get(
            "{}/find_by_id/archival_objects?ref_id[]={}".format(repo, ref_id)
        ).json()
        ao_uri = uri_request["archival_objects"][0]["ref"]
        ao_json = aspace.client.get(ao_uri).json()

    resource_uri = ao_json["resource"]["ref"]
    resource_json = aspace.client.get(resource_uri).json()

    title = etree.Element("{http://purl.org/dc/elements/1.1/}title")
    title_string = ao_json["display_string"]
    find_tags = re.findall("<[^<]+?>", title_string)
    if find_tags:
        for t in find_tags:
            title_string = title_string.replace(t, '"')
    title.text = title_string

    fields.append(title)

    coll_name = etree.Element(
        "{http://purl.org/dc/terms/}isPartOf",
        nsmap={"dcterms": "http://purl.org/dc/terms/"},
    )
    coll_name.text = resource_json["title"]
    fields.append(coll_name)

    call_num = etree.Element("{http://purl.org/dc/elements/1.1/}identifier")
    call_num.text = "{}".format(resource_json["id_0"])
    fields.append(call_num)

    container_id = etree.Element("{http://purl.org/dc/elements/1.1/}source")
    container_id.text = "{} ({})".format(
        get_container_info(ao_json), resource_json["id_0"]
    )
    fields.append(container_id)
    if "language" in ao_json:
        language = etree.Element("{http://purl.org/dc/elements/1.1/}language")
        language.text = ao_json["language"]
        fields.append(language)

    dc_record = etree.Element(
        "{http://purl.org/dc/elements/1.1/}record", nsmap=nsmap)
    for field in fields:
        dc_record.append(field)
    return dc_record
```

- Add crosswalked metadata to `ie_dict`

```python
ie_dict[ie_pid] = {
    "tree": etree.tostring(tree, encoding="utf-8"),
    "ref_id": ref_id,
    "aspace_dc": aspace_dc,
    "caption": caption,
}
```

#### Add Link to ArchivesSpace

#### Update Metadata in Rosetta

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
