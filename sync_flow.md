# Syncing Rosetta with ArchivesSpace or Aleph

## High level code review

### Before Sync

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

### Syncing with Aleph

- Look for IDs in IE Tree using XPath
  `ids = tree.xpath("//dc:identifier[not(@*)]", namespaces=nsmap)`
- Iterate through IDs looking for Aleph numbers
- If Aleph ID found, find Aleph record using Aleph API

```python
def find_aleph_record(aleph_id):
    find_num = re.search("\d{9}", aleph_id)
    if find_num:
        aleph_id = find_num.group(0)
        aleph_xml = get_marcxml_from_aleph(aleph_id)
        if aleph_xml:
            return aleph_xml
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

- Add IE information to `ie_dict`

```python
    ie_dict[ie_pid] = {
        "tree": etree.tostring(tree_copy, encoding="utf-8"),
        "aleph_id": aleph_id,
        "aleph_dc": aleph_dc,
        "caption": caption,
    }
```
