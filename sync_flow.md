# Syncing Rosetta with ArchivesSpace or Aleph - Code Guide

## ArchivesSpace

### 1) Retrieve IE Metadata

- Generate list of IEs from CSV, IE List or SIP
- For each IE in the list
  - Get IE Tree metadata
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
